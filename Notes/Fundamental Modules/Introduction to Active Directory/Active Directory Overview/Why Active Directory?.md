# Why Active Directory?

You will see Active Directory in the vast majority of Windows-based enterprise networks. It is Microsoft’s directory service and is built as a distributed, hierarchical system that allows central control over users, computers, groups, policies, devices, file shares, and trust relationships. From an operational point of view, it simplifies management. From a security point of view, it introduces a very large attack surface.

<img width="2576" height="1862" alt="whyad5" src="https://github.com/user-attachments/assets/da7557a4-0c90-42f5-9988-7bf1eed8dbdb" />

Active Directory is responsible for authentication and authorisation across a Windows domain. Because it has evolved over decades, it was designed with strong backward compatibility in mind. This means that many features are not secure by default and rely heavily on correct configuration. In practice, this often leads to weaknesses that can be abused to move laterally or escalate privileges inside a network.

One critical point you need to understand early is that Active Directory behaves like a large, mostly readable directory. Even a basic domain user account, with no administrative privileges, can query and enumerate a significant portion of the environment. This includes users, groups, computers, and relationships between them. As a result, any compromised user account can be used as a starting point to map the domain and search for misconfigurations or design flaws.

Because of this, securing Active Directory is not just about protecting administrator accounts. Defence-in-depth is essential. You must think in terms of least privilege, proper delegation, network segmentation, and continuous hardening. Many attacks can be performed with nothing more than a standard domain user account, which highlights how dangerous poor defaults and weak controls can be. Real-world examples, such as the noPac vulnerability disclosed in late 2021, demonstrate how seemingly minor issues can be chained into full domain compromise.

From an administrative perspective, Active Directory is powerful and scalable. It is designed to support very large environments, with millions of objects in a single domain and the ability to expand into multiple domains as an organisation grows. This scalability is one of the reasons it remains so widely adopted.

Active Directory is also a prime target simply because of how common it is. A significant majority of large organisations rely on it, which makes it an attractive focus for attackers. If an attacker gains initial access through a phishing email or another low-level compromise, even as a standard user, they often have enough visibility to begin identifying attack paths almost immediately.

As a security professional, you will encounter Active Directory environments throughout your career, ranging from small businesses to global enterprises. Understanding how Active Directory is structured and how it operates is essential whether you are attacking it during an assessment or defending it in production.

Modern ransomware operations frequently target Active Directory as a central component of their attack chains. High-impact vulnerabilities such as PrintNightmare and Zerologon have been actively abused to escalate privileges and move laterally within networks. These attacks show how quickly poor patching and weak design choices can lead to full domain takeover.

If you want to find and prevent these issues before attackers do, you need to understand Active Directory at a fundamental level. New vulnerabilities and misconfigurations are discovered regularly, often requiring little more than standard user access to exploit. While many open-source tools exist to enumerate and attack Active Directory, tools alone are not enough. Their effectiveness depends entirely on your understanding of what you are looking at and why it matters.

This set of notes is intended to give you that foundation. You will work through how Active Directory is structured, how its core components function, how permissions and privileges are applied, and how administrators typically manage it. You will also see how common weaknesses arise and why they persist.

Before you move on to deeper topics such as enumeration techniques, exploitation paths, lateral movement, post-exploitation, and persistence, you need this groundwork. A strong understanding of Active Directory fundamentals will make complex environments easier to reason about and will give you the confidence to approach both small and very large networks with the same level of clarity.

Active Directory is not disappearing any time soon. Even as organisations adopt cloud and hybrid models, on-premises Active Directory remains deeply embedded in most enterprise networks. If you are performing network or internal penetration tests, you should expect to encounter it in almost every engagement.

By building a solid understanding now, you make yourself both a more effective attacker and a more credible defender. This knowledge will also allow you to provide practical, informed remediation advice, rather than surface-level recommendations.
