# Beyond this Module

The skills covered in this module are not academic exercises -- they map directly to what a penetration tester is expected to execute on real engagements. Pivoting and tunneling are not isolated techniques but the connective tissue between initial access and full compromise of an internal network. Every tool and concept covered here has a practical role in the sequence of gaining a foothold, moving laterally, and reaching target assets across segmented networks.

## Real World Application

On an actual engagement, the work from this module feeds into a broader attack chain. Once a pivot point is established and tunnels are running, teammates and senior testers rely on those access paths to conduct their own phases of the assessment. The actions that typically build on what this module covers include:

- Using tunnels to perform further exploitation and lateral movement across subnets that would otherwise be unreachable
- Implanting persistence mechanisms within each subnet accessed to ensure continued access if a session drops
- Establishing command and control channels that route through the pivot infrastructure to reach deeper network segments
- Using tunnels to bring additional tooling into the environment and exfiltrate data while evading security controls that monitor standard traffic paths

A weak understanding of networking fundamentals makes all of this harder than it needs to be. If subnetting, routing, or layer 2 and 3 concepts felt uncertain at any point during this module, the [Introduction to Networking](https://academy.hackthebox.com/course/preview/introduction-to-networking) module on HTB Academy is the right place to fill those gaps before going further.

## Recommended Practice Boxes

The three HTB machines called out here are specifically worth working through because each one demands creative use of tunneling in a different way.

- [Reddish](https://app.hackthebox.com/machines/Reddish) is the most challenging of the three and is the machine where IppSec demonstrates Chisel in detail, including shrinking the Go binary using `ldflags` and `upx` before transfer. His [walkthrough](https://www.youtube.com/watch?v=Yp4oxoQIBAM&t=2466) is one of the best practical references for multi-hop tunneling available publicly
- [Inception](https://app.hackthebox.com/machines/Inception) involves layered pivoting through nested environments and tests the ability to chain tools across multiple hops
- [Enterprise](https://app.hackthebox.com/machines/Enterprise) covers pivoting within a more traditional enterprise network layout

[IppSec's search tool](https://ippsec.rocks/) is genuinely useful here -- it indexes the content of his video walkthroughs so you can search by technique or tool name and jump directly to the relevant timestamp rather than watching full videos from the start.

## Pro Labs for Deeper Practice

Pro Labs are where these skills get stress-tested against realistic corporate network simulations. The three most relevant to pivoting are:

- [Dante](https://app.hackthebox.com/prolabs/overview/dante) -- the most beginner-friendly Pro Lab and the one most directly aligned with this module. Nearly every machine in Dante requires at least one pivot layer, forcing fluency with SSH tunnels, Metasploit routing, proxychains, and Chisel in combination. Multiple reviewers note that double pivoting is a core requirement, not an edge case
- [Offshore](https://app.hackthebox.com/prolabs/overview/offshore) -- intermediate difficulty, heavier on Active Directory with significant pivoting requirements across multiple domains and trust relationships
- [RastaLabs](https://app.hackthebox.com/prolabs/overview/rastalabs) -- intermediate, red team focused, built by RastaMouse who maintains an excellent blog covering C2 infrastructure, payloads, and pivoting in depth

The [Ascension](https://app.hackthebox.com/prolabs/overview/ascension) Mini Pro Lab is flagged as the most challenging option, featuring two separate Active Directory domains with extensive enumeration and attack requirements throughout.

## Blogs and Resources Worth Bookmarking

A handful of external resources referenced in this module are worth keeping accessible during future work:

- [0xdf's blog](https://0xdf.gitlab.io/) covers HTB machine walkthroughs with unusually detailed explanations of why each technique works, not just how -- the Chisel and SSF tunneling post grew directly out of his Reddish writeup and is a practical reference for forward and reverse tunnel configurations
- [SpecterOps SSH tunneling guide](https://posts.specterops.io/offensive-security-guide-to-ssh-tunnels-and-proxies-b525cbd4d4c6) covers SSH tunneling and proxying across multiple protocols in a depth that makes it useful as a reference during live engagements
- [RastaMouse's blog](https://rastamouse.me/) focuses on red teaming, C2 infrastructure, and evasion -- directly applicable to the operational security considerations that come after tunnels are established
- [SANS Pivoting Cheat Sheet webcast](https://www.sans.org/webcasts/dodge-duck-dip-dive-dodge-making-the-pivot-cheat-sheet-119115/) provides a broad overview of pivoting tools and their use cases in a format that is useful for quick review before an engagement
- [Plaintext's Pivoting Workshop](https://youtu.be/B3GxYyGFYmQ) was originally built for the HTB Cyber Apocalypse CTF 2022 and remains one of the most practical and accessible video resources on the topic
