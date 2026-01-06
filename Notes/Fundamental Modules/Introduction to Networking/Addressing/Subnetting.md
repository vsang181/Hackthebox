# Subnetting

**Subnetting** is the process of dividing a larger IPv4 address range into **smaller, logical networks** called subnets. Each subnet groups hosts that share the same network address, making networks easier to manage, route, and secure.

A helpful mental model is to think of subnetting like labelled entrances in a large office building. Each entrance leads to a specific department. People inside one department can move freely, but to reach another department, they must pass through a controlled point (the router) 

---

## What Subnetting Helps You Determine

When you analyse or design a subnet, you should always be able to identify:

* Network address
* Broadcast address
* First usable host
* Last usable host
* Total number of usable hosts

If you cannot quickly derive these values, you will struggle with routing, firewall rules, and scoping during assessments.

---

## Example Subnet

Let us work through a concrete example step by step.

```text
IPv4 Address : 192.168.12.160
Subnet Mask  : 255.255.255.192
CIDR         : /26
```

---

## Network vs Host Portion

The subnet mask acts as a **template** that tells you which bits are fixed (network) and which bits can change (hosts).

### Binary Representation

```text
IP Address  : 11000000.10101000.00001100.10100000
Subnet Mask : 11111111.11111111.11111111.11000000
CIDR        : /26
```

* The **1-bits** in the subnet mask represent the **network portion**
* The **0-bits** represent the **host portion**

Only the host bits are allowed to change.

---

## Network Address

To find the **network address**, set **all host bits to 0**.

```text
Network Address: 192.168.12.128/26
```

This address identifies the subnet itself and cannot be assigned to a host.

---

## Broadcast Address

To find the **broadcast address**, set **all host bits to 1**.

```text
Broadcast Address: 192.168.12.191
```

This address is used to communicate with **all hosts in the subnet** and is also not assignable.

---

## Usable Host Range

Once you know the network and broadcast addresses, everything in between is usable.

| Type       | Address        |
| ---------- | -------------- |
| Network    | 192.168.12.128 |
| First Host | 192.168.12.129 |
| Last Host  | 192.168.12.190 |
| Broadcast  | 192.168.12.191 |

A `/26` subnet provides:

```text
64 total addresses
64 - 2 = 62 usable host addresses
```

---

## Subnetting Into Smaller Networks

Now assume you are asked to split this `/26` subnet into **4 smaller subnets**.

### Step 1: Understand the Math

Subnetting always works in **powers of two**.

| Power | Value |
| ----- | ----- |
| 2²    | 4     |
| 2⁴    | 16    |
| 2⁶    | 64    |

To create **4 subnets**, you need **2 additional network bits** (`2² = 4`).

---

### Step 2: Extend the Subnet Mask

```text
Original CIDR : /26
New CIDR      : /28
Subnet Mask   : 255.255.255.240
```

Each `/28` subnet now contains:

```text
16 total addresses
14 usable hosts
```

---

### Step 3: Calculate the New Subnets

Starting from the original network address and incrementing by **16** each time:

| Subnet | Network        | First Host | Last Host | Broadcast | CIDR |
| ------ | -------------- | ---------- | --------- | --------- | ---- |
| 1      | 192.168.12.128 | .129       | .142      | .143      | /28  |
| 2      | 192.168.12.144 | .145       | .158      | .159      | /28  |
| 3      | 192.168.12.160 | .161       | .174      | .175      | /28  |
| 4      | 192.168.12.176 | .177       | .190      | .191      | /28  |

This is a predictable pattern once you recognise the block size.

---

## Mental Subnetting (Fast Method)

Subnetting looks math-heavy at first, but it becomes quick once you internalise a few rules.

---

### Step 1: Identify Which Octet Changes

Use these breakpoints:

| CIDR | Changing Octet |
| ---- | -------------- |
| /8   | 1st            |
| /16  | 2nd            |
| /24  | 3rd            |
| /32  | 4th            |

Example:

```text
192.168.1.1/25
```

Only the **fourth octet** can change.

---

### Step 2: Calculate the Block Size

Use the modulo operation:

```text
CIDR % 8
```

Example:

```text
25 % 8 = 1
```

This means:

* 1 bit is used in the current octet
* Block size = 2^(8−1) = 128

---

### Common Block Sizes

| Remainder | Block Size |
| --------- | ---------- |
| 0         | 256        |
| 1         | 128        |
| 2         | 64         |
| 3         | 32         |
| 4         | 16         |
| 5         | 8          |
| 6         | 4          |
| 7         | 2          |

If you forget the powers of two, just keep dividing **256 by 2** until you reach the remainder.

---

### Step 3: Determine the Address Range

Remember:

* **0 is a valid address**, not null
* First address = network
* Last address = broadcast

For `/25`:

```text
First range : 192.168.1.0 – 127
Usable      : 192.168.1.1 – 126
```

If the IP is above 128:

```text
Second range: 192.168.1.128 – 255
Usable      : 192.168.1.129 – 254
```

---

## Why Subnetting Matters to You

Subnetting directly impacts:

* Scan scope and accuracy
* Reachability of hosts
* Pivoting paths
* Firewall and routing behaviour
* Blind spots during assessments

Misunderstanding subnetting leads to **false assumptions**, missed assets, and incorrect conclusions.

---

## Key Takeaways

* Subnetting divides networks into smaller logical units
* Subnet masks define network vs host bits
* Network and broadcast addresses are never usable
* Everything works in powers of two
* Block size determines address ranges
* Mental subnetting saves time during real engagements

Once subnetting clicks, IP ranges stop feeling arbitrary and start feeling **structured, predictable, and intentional**.
