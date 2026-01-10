# Active Directory Research Over the Years

Active Directory has been heavily studied by the security community for many years, and much of what you will use today is the result of research that started gaining serious momentum around the early 2010s. From that point onwards, researchers began uncovering fundamental weaknesses, design assumptions, and misconfigurations that attackers could reliably exploit. Many of the attack techniques you will learn now are refinements or extensions of work first published during this period.

As you go through this material, it is important to understand that Active Directory remains a moving target. Its size, complexity, and continued evolution mean that new attack paths are still being discovered. The same applies to Azure AD and hybrid environments, which expand the attack surface even further. Both attackers and defenders must keep up with new research, tools, and exploitation techniques to stay effective.

The timeline below highlights a selection of influential attacks, tools, and research that shaped how Active Directory is assessed today. This is not an exhaustive history, but it gives you context for why certain techniques exist and why some tools are considered essential. Many of these discoveries came from independent researchers whose work fundamentally changed how we think about securing Windows domains.

A key takeaway is that critical flaws continue to appear. Even relatively recent discoveries have shown that full domain compromise can sometimes be achieved from a standard user account under the right conditions. As you study Active Directory, you should assume that new techniques will continue to emerge and that staying current is part of the job.

# AD Attacks and Tools Timeline

## 2021

Several high-impact Active Directory issues came to light in 2021. PrintNightmare exposed a remote code execution flaw in the Windows Print Spooler that could be abused to compromise systems across a domain. Shadow Credentials techniques demonstrated how low-privileged users could impersonate other users or computer accounts when specific conditions were met, enabling privilege escalation.

Later in the year, the noPac attack was disclosed while much of the industry was focused elsewhere. This attack showed how chaining specific misconfigurations could allow a standard domain user to gain full domain control, reinforcing how dangerous weak defaults and legacy behaviour can be.

## 2020

The Zerologon vulnerability emerged as one of the most severe Active Directory flaws seen in years. It allowed attackers to impersonate unpatched domain controllers, effectively breaking the trust model at the core of a Windows domain.

## 2019

Research during this year refined existing techniques rather than introducing entirely new classes of attacks. Work on Kerberoasting demonstrated new approaches to abusing service accounts. Resource-based constrained delegation gained attention as researchers showed how it could be abused for privilege escalation when poorly configured. This period also saw major updates to offensive frameworks, improving stability and extensibility.

## 2018

Several important concepts and tools appeared in 2018. The Printer Bug highlighted how authentication coercion could be abused using RPC interfaces. Kerberos-focused tooling matured significantly, making ticket-based attacks more accessible. Research into forest trusts demonstrated that trust boundaries were weaker than many administrators assumed. Tools for auditing Active Directory security posture also became more widely adopted during this time.

## 2017

This year introduced techniques for attacking accounts that did not require Kerberos pre-authentication, expanding the attack surface further. Research into access control list abuse showed how subtle permission misconfigurations could lead to powerful escalation paths. Trust relationships between domains also received deeper scrutiny, revealing additional weaknesses.

## 2016

A major shift occurred with the release of graph-based analysis tools that allowed attackers and defenders to visualise complex attack paths within Active Directory environments. This fundamentally changed how large domains were assessed and understood.

## 2015

Many foundational tools used today were first released around this time. Techniques for synchronising password data from domain controllers were demonstrated, along with frameworks that automated large portions of Active Directory enumeration and exploitation. Research into delegation settings revealed how dangerous certain configurations could be when misunderstood. Tooling released during this period remains central to most internal network assessments.

## 2014

Early reconnaissance frameworks and Kerberos abuse techniques were publicly demonstrated, laying the groundwork for much of the research that followed. These discoveries showed how service account design choices could be turned into reliable attack vectors.

## 2013

Network poisoning attacks targeting name resolution protocols became more practical with the release of tooling that automated interception and credential capture. These techniques proved highly effective in internal environments and continue to be relevant today, especially in poorly segmented networks.

As you work through Active Directory concepts and attacks, keep this timeline in mind. Many techniques build directly on earlier discoveries, and understanding their origins will help you recognise why they still work. More importantly, it reinforces a critical lesson: Active Directory security is not a solved problem, and continuous learning is required to defend it properly.
