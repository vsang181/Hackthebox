# Pivoting, Tunneling & Port Forwarding

Pivoting, tunneling, and port forwarding are three closely related but distinct techniques used during penetration testing to extend access beyond the initially compromised system. Together they form the foundation of post-exploitation network movement.

## Table of Contents

1. Introduction

    - Introduction to Pivoting, Tunneling, and Port Forwarding
    - The Networking Behind Pivoting

2. Choosing The Dig Site & Starting our Tunnels

    - Dynamic Port Forwarding with SSH and SOCKS Tunneling
    - Remote/Reverse Port Forwarding with SSH
    - Meterpreter Tunneling & Port Forwarding

3. Playing Pong with Socat

    - Socat Redirection with a Reverse Shell
    - Socat Redirection with a Bind Shell

4. Pivoting Around Obstacles

    - SSH for Windows: plink.exe
    - SSH Pivoting with Sshuttle
    - Web Server Pivoting with Rpivot
    - Port Forwarding with Windows Netsh

5. Branching Out Our Tunnels

    - DNS Tunneling with Dnscat2
    - SOCKS5 Tunneling with Chisel
    - ICMP Tunneling with SOCKS

6. Double Pivots

    - RDP and SOCKS Tunneling with SocksOverRDP
      
7. Additional Considerations

    - Detection & Prevention
    - Beyond this Module
