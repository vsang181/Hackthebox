# Introduction to Pivoting, Tunneling, and Port Forwarding

This section draws a clear line between three concepts that get conflated regularly in the industry. Understanding the distinction matters in practice because each serves a different purpose and is applied at different stages of an engagement.

## The Three Concepts Defined

**Lateral movement** is about spreading across a network you already have access to. It is wide rather than deep -- using credentials, hashes, or trust relationships to move between hosts within the same network segment, often with the goal of elevating privileges along the way. MITRE ATT&CK catalogues it under [TA0008](https://attack.mitre.org/tactics/TA0008/) as a distinct tactic covering techniques like Pass-the-Hash, RDP hijacking, and SMB-based execution. The key characteristic is that the target hosts are reachable -- the challenge is authentication and privilege, not network access.

**Pivoting** is about depth rather than breadth. It is used specifically to cross a network boundary that would otherwise block you -- a firewall rule, a VLAN separation, an air gap between enterprise and operational networks. A pivot host is any compromised machine that sits at the boundary between two segments and can relay traffic between them. The host does not need to be powerful or highly privileged -- it just needs connectivity to both sides. Common terms for the pivot host include jump host, foothold, beach head, or proxy depending on the context and organisation.

**Tunneling** is about obfuscation. It encapsulates one protocol's traffic inside another to make it appear benign. A C2 channel hidden inside HTTPS GET and POST requests looks like normal web browsing to a firewall or a SIEM analyst skimming logs. SSH traffic wrapping arbitrary TCP connections looks like a legitimate admin session. The goal is to avoid detection for as long as possible, not just to reach a destination.

## A Useful Analogy

The module's stuffed toy analogy captures tunneling well. The key (your actual payload or C2 traffic) is hidden inside a toy (HTTP or SSH). Anyone inspecting the package at the border (a firewall or IDS) sees a toy. Only the intended recipient knows what is inside and how to use it. The same principle applies to VPNs -- they are simply a productised, legitimate application of the same tunneling concept.

## How They Work Together in Practice

| Concept | Direction | Primary Challenge | Example |
|---|---|---|---|
| Lateral movement | Wide -- across hosts in reach | Authentication and privileges | Reusing local admin credentials across three Windows hosts in the same subnet |
| Pivoting | Deep -- into unreachable segments | Network segmentation | Using a dual-homed engineering workstation to reach an OT/operational network from the enterprise side |
| Tunneling | Evasive -- blending into allowed traffic | Detection and filtering | Wrapping Meterpreter callbacks inside HTTPS to pass through an egress firewall that blocks unknown ports |

## First Steps on a New Host

The module highlights that when landing on a host for the first time, three things should be checked immediately:

- **Privilege level** -- higher privileges make pivoting easier and open more techniques
- **Network connections and adapters** -- a host with more than one NIC is a strong indicator of dual-homed access to multiple segments
- **VPN or remote access software** -- an existing remote access tool may already provide a path to another network without needing to build your own tunnel

A dual-homed host is the most valuable pivot candidate because it has native connectivity to both sides. Without identifying it, you may spend significant time trying to route through a host that simply cannot reach your intended target. With it, the network boundary that was designed to stop lateral movement becomes the exact reason the host is useful to you as an attacker.
