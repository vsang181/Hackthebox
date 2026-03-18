# ICMP Tunneling with SOCKS

ICMP tunneling works by encapsulating arbitrary TCP data inside ICMP echo request and reply packets. Most firewalls are configured to block TCP and UDP ports selectively, but permit ICMP because ping is considered a basic network diagnostic tool. This oversight creates a covert channel where data travels disguised as ordinary ping traffic, completely bypassing port-based firewall rules.

[Ptunnel-ng](https://github.com/utoni/ptunnel-ng) is the tool used here to create this channel. It establishes a reliable TCP-over-ICMP tunnel between the attack host and the pivot host, and once that channel is open, any TCP-based traffic including SSH, proxychains, and dynamic port forwarding can flow through it.

## Building and Deploying Ptunnel-ng

Clone the repo and run the `autogen.sh` build script:

```bash
git clone https://github.com/utoni/ptunnel-ng.git
sudo ./autogen.sh
```

If you need a static binary that avoids glibc version conflicts on the target host, modify the autogen script before running it:

```bash
sudo apt install automake autoconf -y
cd ptunnel-ng/
sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
./autogen.sh
```

A statically linked binary carries all its dependencies internally and will run regardless of what library versions are installed on the pivot host. Transfer the repo after building:

```bash
scp -r ptunnel-ng ubuntu@10.129.202.64:~/
```

## Starting the ICMP Tunnel

On the pivot host, start the ptunnel-ng server. The `-r` flag specifies the IP the server will accept connections on, and `-R` specifies the internal port to forward to -- in this case SSH on port 22:

```bash
ubuntu@WEB01:~/ptunnel-ng/src$ sudo ./ptunnel-ng -r10.129.202.64 -R22

[inf]: Starting ptunnel-ng 1.42.
[inf]: Forwarding incoming ping packets over TCP.
[inf]: Ping proxy is listening in privileged mode.
[inf]: Dropping privileges now.
```

On the attack host, connect to the ptunnel-ng server. The `-p` flag points to the pivot host, `-l` specifies the local port to listen on for the incoming SSH connection, and `-r/-R` match the server configuration:

```bash
sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22

[inf]: Relaying packets from incoming TCP streams.
```

At this point the ICMP tunnel is live. All traffic sent to `127.0.0.1:2222` on the attack host is encapsulated in ICMP packets, sent to the pivot host, and delivered to port 22 on the other side.

## SSH Through the ICMP Tunnel

With the tunnel active, connect via SSH to local port 2222 as if connecting to the pivot directly:

```bash
ssh -p2222 -lubuntu 127.0.0.1
```

A successful SSH session confirms the tunnel is working end to end. Both the ptunnel-ng server and client will display session statistics once connected:

```
[inf]: Incoming tunnel request from 10.10.14.18.
[inf]: Starting new session to 10.129.202.64:22 with ID 20199

Session statistics:
[inf]: I/O:   0.00/  0.00 mb ICMP I/O/R:  248/22/0  Loss:  0.0%
```

The packet counters confirm data is moving through ICMP rather than TCP, and a loss rate of 0.0% shows the tunnel is stable.

## Layering Dynamic Port Forwarding on Top

The ICMP tunnel only wraps SSH so far, but SSH's `-D` flag can then add a SOCKS proxy layer on top of it, creating a tunnel within a tunnel:

```bash
ssh -D 9050 -p2222 -lubuntu 127.0.0.1
```

Update `/etc/proxychains.conf` to point to port 9050, and full network access to the `172.16.5.0/23` network becomes available through proxychains:

```bash
proxychains nmap -sV -sT 172.16.5.19 -p3389
```

Output:

```
PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

The data path here is worth appreciating in full: proxychains sends TCP traffic to the local SOCKS proxy on 9050, SSH wraps it and sends it through local port 2222, ptunnel-ng encapsulates that in ICMP echo packets, the pivot host decapsulates them and delivers the original SSH traffic to port 22, and SSH on the pivot forwards the inner traffic to the target.

## What Wireshark Reveals

The Wireshark comparison in the material makes the detection difference concrete. A direct SSH connection to the pivot shows TCP handshakes and SSHv2 packets. The exact same SSH session through the ptunnel-ng ICMP tunnel shows only ICMP echo request and reply packets with no TCP visible at the capture level. To a network defender or IDS rule watching for SSH specifically, the session is invisible.

The one indicator that does stand out is the ICMP packet size. Standard ping echo requests carry 32 to 64 bytes of data. ICMP packets carrying tunneled SSH traffic will carry significantly more data in the payload field, and their frequency will be far higher than normal ping usage. Writing detection rules that flag high-volume ICMP or ICMP packets with anomalously large data sections will catch ptunnel-ng traffic even when port-based inspection cannot.
