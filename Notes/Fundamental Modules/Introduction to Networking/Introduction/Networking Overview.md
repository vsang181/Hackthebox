# Networking Overview

A **network** exists to allow two or more computers to communicate. While this sounds simple, real-world networks quickly become complex once you introduce different **topologies**, **media**, and **protocols**. As a security professional, you need to understand these fundamentals because networking failures are often **silent**—things break without obvious errors, and that is exactly where attackers thrive 

You will encounter networks built using:

* Different topologies (star, mesh, tree)
* Different transmission media (Ethernet, fibre, coaxial, wireless)
* Different protocols (TCP, UDP, IP, and legacy protocols)

If you do not understand how these pieces fit together, you will miss issues that are not immediately visible.

---

## Flat Networks vs Segmented Networks

Creating a large, flat network is relatively easy. From an operational perspective, it can work perfectly fine. From a security perspective, it is a poor design choice.

A flat network is like building a house and assuming it is secure because the front door has a lock.

When you divide a network into **smaller, purpose-built segments** and control how they communicate, you add defensive layers. Pivoting across a segmented network is still possible, but doing it **quickly and quietly** becomes much harder. That slowdown is often enough to trigger detection or stop an attack entirely.

---

## Thinking in Physical Security Analogies

Using physical security analogies helps you reason about network design.

### Example 1: Fences (Access Control Lists)

Placing **Access Control Lists (ACLs)** between networks is like putting a fence around a property.

Yes, someone *could* jump the fence, but:

* It is unusual
* It stands out
* It is easier to detect

If you see a printer network communicating with servers over HTTP, you should immediately ask why.

---

### Example 2: Lighting (Documentation and Visibility)

Clearly documenting the purpose of each network is like installing lights around a property.

Good visibility allows you to spot activity that does not belong. For example:

* Why is the printer network talking to the internet at all?

If you cannot answer that question quickly, the design is already failing.

---

### Example 3: Bushes and Deterrents (IDS/IPS)

Deterrents such as bushes under windows make attacks less attractive.

In networking, **Intrusion Detection Systems** like Suricata or Snort serve a similar purpose. They do not prevent all attacks, but they raise the risk for the attacker.

If you detect a port scan originating from a printer network, something is very wrong.

---

## Why Flat Networks Cause Real Problems

These examples may sound obvious, but in practice they are ignored all the time—especially in flat `/24` networks where every device receives an IP address via DHCP and can talk to everything else by default.

Once everything lives in the same broadcast domain, enforcing meaningful restrictions becomes difficult.

---

## Story Time: A Penetrester’s Oversight

Most networks use a `/24` subnet, and many penetration testers assume this without verifying it.

A `/24` network (`255.255.255.0`) allows all hosts with the same first three octets to communicate freely.

A `/25` subnet splits that range in half.

Here is a real-world-style example:

* Server Gateway: `10.20.0.1/25`
* Domain Controller: `10.20.0.10/25`
* Client Gateway: `10.20.0.129/25`
* Client Workstation: `10.20.0.200/25`
* Pentester IP: `10.20.0.252/24` (gateway set to `10.20.0.1`)

The tester successfully attacked client workstations and even stole credentials. They believed they had done a great job.

In reality, they never left the **client network**.

The Domain Controller was not offline—it was simply unreachable due to incorrect subnetting. High-value targets were missed entirely because the network design was not understood.

If this feels confusing now, that is fine. By the end of this module, it should make perfect sense.

---

## Networking as Mail Delivery

You can think of networking as **mail delivery**.

* One computer sends a package
* Another computer receives it
* The package must have a destination address

In networking, that address is usually a **Fully Qualified Domain Name (FQDN)** or an **IP address**.

---

## FQDN vs URL

It is important to distinguish between these two:

* An **FQDN** (for example, `www.example.com`) identifies the *building*
* A **URL** (for example, `https://www.example.com/floor2/office5`) identifies the *building, floor, room, and mailbox*

You will explore this distinction in more detail later, but the mental model is useful early on.

---

## The Role of the Router and ISP

When you send traffic from your home network to a company website:

* Your **router** acts like your local post office
* Your **ISP** acts like the main sorting facility

The ISP uses **DNS** as its address book, translating names into IP addresses. Once the destination is known, packets are routed directly to the target network.

The response follows the same path back, using your IP address as the return address.

---

## Designing Networks Properly (Extra Points)

If you look at a typical home-to-company diagram, a secure design should actually consist of **multiple separate networks**, not one big one.

At a minimum:

* **Web servers** belong in a **DMZ**
  They accept connections from the internet and are more likely to be compromised. Isolation limits damage.

* **Workstations** should be isolated from each other
  Ideally, host-based firewalls prevent workstation-to-workstation communication.

* **Network infrastructure** (switches and routers) should live on an **administration network**
  This prevents attackers from spoofing routing protocols such as OSPF.

* **IP phones** should be on their own network
  This prevents eavesdropping and allows traffic prioritisation to reduce latency.

* **Printers** should *always* be isolated
  Printers are notoriously insecure and frequently trigger NTLM authentication, which can leak credentials.

---

## A Real Attack Scenario

During a real engagement, a penetration tester exploited a printer to establish a reverse shell. The printer was shipped to the target organisation under the guise of helping staff work more efficiently during COVID.

The printer was plugged in almost immediately.

Because of poor network design:

* The printer could reach the internet
* Workstations could talk to the printer over SMB
* The printer could initiate connections to clients

A domain administrator’s workstation authenticated to the printer, handing over credentials.

This attack relied entirely on **network design failures**, not exotic exploits.

---

## What You Should Take Away

* Networking mistakes are often silent
* Flat networks are easy but dangerous
* Segmentation slows attackers down
* Subnetting mistakes cause blind spots
* Printers are high-risk devices
* Design matters more than tools

As you continue, always ask yourself:

**“Should these two things really be able to talk to each other?”**

That question alone will uncover more security issues than most scanners ever will.
