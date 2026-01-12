# Oracle TNS

Oracle Transparent Network Substrate (TNS) is the **network protocol layer used by Oracle Net Services** to let Oracle database clients and applications communicate with Oracle databases over a network. In most environments you will see it exposed via the **Oracle Listener**, which by default listens on **TCP/1521** (though this is configurable).

TNS is common in large enterprise deployments (finance, healthcare, retail) because it supports:

* Name resolution (service/SID mapping)
* Connection management and session handling
* Load balancing and failover (service-based routing)
* Security features (encryption options, access controls, TLS/SSL support depending on configuration)

---

## Key Files and Concepts

### `tnsnames.ora` (client-side)

Client-side mapping of a friendly service name (for example `ORCL`) to a host, port, and service name or SID.

Example:

* `HOST` and `PORT` tell the client where the listener is.
* `SERVICE_NAME` or `SID` tells Oracle which database service/instance to connect to.

### `listener.ora` (server-side)

Listener configuration defining:

* Which addresses (IP/hostname, port, protocol) it listens on
* Which services/SIDs it knows about
* Additional behaviour (logging, security, etc.)

### SID vs SERVICE_NAME

* **SID**: identifies a specific database instance (more “instance-centric”)
* **SERVICE_NAME**: service-based connection name (common in modern deployments and RAC)

---

## Default and “Worth-Remembering” Oracle Details

* Oracle Listener default port: **1521/tcp**
* Legacy Oracle versions sometimes shipped with well-known defaults (for example, old “install-time” defaults and sample accounts like `scott/tiger` in lab setups).
* The **DBSNMP** account is historically notable and sometimes left with weak credentials in poorly managed environments.

---

## Footprinting Oracle TNS (Safe Recon)

### 1) Confirm the listener is present

Typical approach is to scan **1521/tcp** and identify the listener banner/version.

Example pattern you often see:

* “Oracle TNS listener 11.x / 12.x / 19c …”

### 2) Enumerate service names / SIDs

If the listener responds, you may be able to identify valid SIDs or services. Common approaches include:

* Nmap Oracle scripts (for example, SID guessing scripts)
* Oracle-aware enumeration tools (for example ODAT in lab environments)

The goal is simply: **find a valid target service identifier** so a client can attempt authentication.

---

## Tooling Notes (ODAT + Instant Client)

In a lab or authorised assessment environment, ODAT (Oracle Database Attacking Tool) is often used to automate enumeration (service discovery, user/password checks, misconfig checks). It typically relies on Oracle client libraries (Instant Client / `sqlplus`) being installed correctly.

A quick sanity check after setup:

* Running `odat.py -h` should display available modules
* `sqlplus` should run without missing library errors (common issue is missing/incorrect `LD_LIBRARY_PATH` / `ldconfig` configuration)

---

## Post-Access Enumeration (High-Level, Non-Destructive)

Once authenticated (even as a low-privileged user), typical “read-only” checks include:

* Listing accessible tables/views:

  * Useful for understanding what data the user can reach
* Checking assigned roles:

  * Helps determine privilege level and possible escalation paths

If a user can connect **“as sysdba”**, that is effectively administrative access and typically indicates:

* Misconfiguration
* Over-privileged account
* Or OS/authentication mapping behaviours in specific setups
