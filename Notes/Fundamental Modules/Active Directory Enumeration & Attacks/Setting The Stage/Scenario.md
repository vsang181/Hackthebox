# Scenario

This scenario sets up the context for the entire module: we are junior penetration testers at CAT-5 Security, given our first largely unsupervised engagement and tasked with demonstrating competency across the full AD attack lifecycle. The scoping document and tasking email are not just set dressing -- they represent the kind of documentation you will encounter and be responsible for interpreting on real engagements.

## Reading the Scoping Document

Before a single packet is sent, the scoping document defines the legal and operational boundaries of the assessment. Getting this wrong has real consequences. Testing systems outside the defined scope, using prohibited techniques, or disrupting production systems can expose the firm to legal liability and immediately ends the professional relationship with the client. The habit of thoroughly reading and understanding scope documentation before doing anything is one of the most important professional practices a penetration tester can build.

For this engagement, the scope breaks down into three areas:

**In scope:**
- `INLANEFREIGHT.LOCAL` - the primary domain including AD and web services
- `LOGISTICS.INLANEFREIGHT.LOCAL` - a subdomain of the primary domain
- `FREIGHTLOGISTICS.LOCAL` - a subsidiary with an external forest trust to INLANEFREIGHT.LOCAL
- `172.16.5.0/23` - the internal subnet containing the target hosts

**Explicitly out of scope:**
- Any subdomains of INLANEFREIGHT.LOCAL not listed above
- Any subdomains of FREIGHTLOGISTICS.LOCAL
- Phishing and social engineering
- Any active attacks against the real-world inlanefreight.com website

The forest trust between INLANEFREIGHT.LOCAL and FREIGHTLOGISTICS.LOCAL is worth paying attention to immediately. Trust relationships between domains and forests are one of the most commonly abused privilege escalation paths in AD environments, and the fact that this one is explicitly in scope signals that cross-forest attacks are expected to be part of the assessment path.

## The Two Assessment Types

The module structures the learning around two distinct internal assessment scenarios that mirror common client requests:

The first simulates starting from an external breach position. The tester begins with no credentials and no foothold, working from an anonymous position on the internal network and simulating an attacker who has already bypassed the perimeter. This models the realistic scenario where someone gains initial access through phishing, a vulnerable public-facing application, or a VPN credential compromise. The goal is to start from nothing and work up to domain compromise through enumeration, credential attacks, and lateral movement.

The second places the attacker directly inside the network with an attack box already on the internal segment. This models either a malicious insider or a scenario where the client wants to understand the blast radius of an internal compromise. Starting with internal network access removes the initial foothold problem and lets the assessment focus entirely on what can be achieved once an attacker is already inside.

Both scenarios share the same end goal: compromise of all in-scope internal domains, with full documentation of every attack path discovered along the way.

## The Authorized Methods

Three categories of activity are explicitly authorized in the scoping document, and each carries important operational constraints:

**External passive enumeration** is authorized without any restrictions on what OSINT sources can be used, but active scanning or exploitation of real-world internet-facing systems is prohibited. This is a standard restriction that protects the client from unintended disruption and protects the tester from legal exposure. The keyword is passive -- search engines, LinkedIn, certificate transparency logs, and similar sources are fair game.

**Internal testing** is authorized from an untrusted insider position with no advance information beyond the scoping document. The instruction that computer systems and network operations must not be intentionally interrupted is significant. This rules out destructive payloads, ransomware simulation, or any technique that could cause downtime. On real engagements this constraint often extends to caution around tools that create noisy network traffic or stress test services.

**Password testing** is explicitly authorized, including offline cracking of captured hashes. The data handling clause -- that captured passwords will not be shared outside the assessment team and will be stored securely -- reflects real professional and contractual obligations. Credential data captured during an engagement is sensitive material that must be handled with the same care as any other confidential client data.

## Why This Documentation Matters

Scoping documents and Rules of Engagement serve multiple purposes simultaneously. They protect the client by defining what disruption risks are acceptable. They protect the tester by establishing documented authorization for actions that would otherwise be illegal. They protect the firm by creating a paper trail that demonstrates professional conduct. And they focus the assessment by forcing both parties to agree on what success looks like before work begins.

A common mistake for newer testers is treating scope documentation as a formality to skim before starting. In practice, it is the document you return to whenever you are uncertain whether a particular technique or target is authorized. Getting a verbal confirmation from a client contact is not sufficient -- the written scoping document is what matters if a question arises later.

With the scope established, the assessment begins with passive external enumeration before any interaction with the internal network. Starting from the outside and building inward is the methodologically sound approach because it mirrors how a real attacker would approach the target, and because information gathered externally often directly enables the internal phases that follow.
