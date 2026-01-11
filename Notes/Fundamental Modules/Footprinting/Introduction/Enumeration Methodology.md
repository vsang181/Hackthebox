# Enumeration Methodology

Complex processes require a **standardised methodology** to help maintain orientation and avoid accidentally omitting critical aspects. Given the wide variety of systems and configurations that targets may present, it is often unpredictable how an approach should be structured. As a result, many penetration testers rely on personal habits and familiar workflows. While effective, this is not a true methodology but rather an **experience-based approach**.

Penetration testing—and enumeration in particular—is a **dynamic process**. To account for this, a **static enumeration methodology** has been developed for both external and internal penetration tests. This methodology allows flexibility while maintaining structure and is designed to adapt to different environments.

The methodology consists of **six layers**, which can be visualised as boundaries that must be crossed during enumeration. Metaphorically, each layer represents a barrier separating the tester from deeper access.

The overall enumeration process can be grouped into three levels:

- **Infrastructure-based enumeration**
- **Host-based enumeration**
- **OS-based enumeration**

<img width="2622" height="1474" alt="enum-method33" src="https://github.com/user-attachments/assets/444056de-5ce3-4d11-97c6-3945924867dd" />

> **Note:** The components shown for each layer represent the main categories only and not an exhaustive list. Additionally, the first and second layers (Internet Presence and Gateway) do not fully apply to internal intranet environments such as Active Directory. Internal infrastructure layers are covered separately in other modules.

---

## Understanding the Layered Model

Think of each layer as a wall or obstacle. The objective is not to break through blindly but to **observe, analyse, and identify gaps**—such as entrances, weak points, or paths around the obstacle.

Although it is sometimes possible to force entry, this often leads to wasted effort if that point does not connect meaningfully to the next layer. Methodical observation almost always yields better results than brute force.

---

## Enumeration Layers Overview

| Layer | Name | Description | Information Categories |
|-----:|------|-------------|------------------------|
| 1 | Internet Presence | Identification of external presence and publicly accessible infrastructure | Domains, Subdomains, vHosts, ASN, Netblocks, IP Addresses, Cloud Instances, Security Measures |
| 2 | Gateway | Identification of security measures protecting external and internal infrastructure | Firewalls, DMZ, IDS/IPS, EDR, Proxies, NAC, Network Segmentation, VPN, Cloudflare |
| 3 | Accessible Services | Identification of externally or internally accessible services | Service Type, Functionality, Configuration, Port, Version, Interface |
| 4 | Processes | Identification of processes handling data and service execution | PID, Processed Data, Tasks, Source, Destination |
| 5 | Privileges | Identification of permissions and access levels | Groups, Users, Permissions, Restrictions, Environment |
| 6 | OS Setup | Identification of operating system configuration and internal setup | OS Type, Patch Level, Network Configuration, OS Environment, Configuration Files, Sensitive Files |

> **Important:** The human factor and employee-derived OSINT have been intentionally excluded from the *Internet Presence* layer for simplicity.

---

## The Labyrinth Analogy

A penetration test can be imagined as a **labyrinth**. Along the way, you will encounter multiple gaps or vulnerabilities. However, not every gap leads deeper into the system.

Penetration tests are time-bound. Even after weeks of testing, it is impossible to state with absolute certainty that no vulnerabilities remain. Attackers who observe and analyse an organisation over months may achieve deeper insight than a short-term assessment allows.

A real-world example of this is the **SolarWinds attack**, where long-term reconnaissance and careful planning enabled deep compromise. This further reinforces the need for a structured methodology that avoids reliance on brute force or chance.

---

## Layer-by-Layer Breakdown

### Layer 1: Internet Presence

This is the starting point for external enumeration. The objective is to identify all potential targets that exist on the Internet.

Activities in this layer include identifying:

- Domains and subdomains  
- IP ranges and netblocks  
- Cloud infrastructure  
- Externally exposed services  

This layer is especially critical when the engagement scope allows the discovery of additional assets beyond predefined targets.

**Goal:** Identify all possible externally reachable systems and interfaces.

---

### Layer 2: Gateway

This layer focuses on understanding **how targets are protected and positioned** within the network.

Here, you examine:

- Security controls  
- Network placement  
- Filtering and monitoring mechanisms  

Due to its complexity and variability, this layer is explored in greater detail in other modules.

**Goal:** Understand what defensive mechanisms exist and how they may affect testing.

---

### Layer 3: Accessible Services

At this stage, each reachable system is examined for the services it exposes.

Each service exists for a specific purpose and offers defined functionality. To interact effectively—and eventually exploit them—you must understand:

- Why the service exists  
- How it operates  
- What data it processes  

This is the **primary focus** of enumeration in this module.

**Goal:** Understand service functionality well enough to communicate with and exploit it effectively.

---

### Layer 4: Processes

Whenever a service executes an action, it processes data. These actions spawn processes that have:

- A source  
- A destination  
- A defined task  

Understanding how data flows through these processes allows you to identify dependencies and trust relationships.

**Goal:** Identify and understand process relationships and data flow.

---

### Layer 5: Privileges

Every service runs under a specific user context with assigned permissions.

Misconfigured privileges often provide unexpected capabilities, particularly in environments like:

- Active Directory  
- Multi-role administrative systems  

These misconfigurations are frequently overlooked by administrators.

**Goal:** Identify what actions are possible within the assigned privilege boundaries.

---

### Layer 6: OS Setup

This layer focuses on the internal operating system configuration, typically after gaining some level of access.

Information gathered here includes:

- OS version and patch level  
- Configuration files  
- Network setup  
- Sensitive local data  

This layer provides insight into the organisation’s internal security maturity and administrative practices.

**Goal:** Understand how systems are managed and what sensitive information is exposed internally.

---

## Enumeration Methodology in Practice

A methodology is **not a step-by-step checklist**. Instead, it is a structured way of thinking and approaching a problem.

In the context of enumeration, the methodology defines **what to explore**, not **which tools to use**. Tools and commands are interchangeable and evolve over time. They are better treated as **reference material or cheat sheets**, not as part of the methodology itself.

Different tools may produce different results, but the objective remains constant:  
**systematically understand the target environment to identify realistic paths to compromise.**
