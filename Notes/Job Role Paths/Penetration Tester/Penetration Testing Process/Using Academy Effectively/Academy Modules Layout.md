# Academy Modules Layout

These notes explain how the Academy modules are structured and, more importantly, **why** they are organised this way. Understanding this early will help you avoid frustration later and give you a clearer mental model of how all the pieces fit together 

Hack The Box did not originally exist as a beginner-friendly platform. It started as a competitive capture-the-flag environment where you were expected to attack targets with no guidance, no hints, and no walkthroughs. That approach works well if you already know what you are doing, but it can be overwhelming if you are just starting your journey in IT or security.

HTB Academy was created to bridge that gap. The idea is to give you structured, guided learning while still preparing you to operate independently later. The Academy is designed to support beginners, while also allowing more experienced practitioners to sharpen specific skills or fill in knowledge gaps. Alongside this, the main HTB platform and Starting Point exist to help you gradually transition from guided learning to fully independent problem-solving.

Everyone approaches learning differently, and this style will not suit everyone. What you see here reflects the perspective of practitioners who have spent years working in IT and security and understand the challenges of moving from theory to real-world capability.

## Why the Foundations Matter

Information Technology is a broad field made up of many specialised areas such as networking, systems administration, development, databases, and security. Cyber security, and penetration testing in particular, sits on top of all of these disciplines.

You do not need to be an expert in every area, but the more you understand how systems actually work, the more effective you will be as a penetration tester. You cannot confidently assess a system if you do not understand what “normal” looks like. Attacks almost always succeed because of small mistakes, misconfigurations, or incorrect assumptions made by humans.

For example, a developer may have years of experience building web applications, yet a single overlooked error can expose sensitive data or bring down an entire service. Your role as a tester is to identify those weak points and demonstrate their impact. That requires technical depth, not just tool usage.

This is why the Academy material may feel demanding at first. The structure is intentional. Over time, you will likely realise that revisiting fundamentals repeatedly is the fastest way to make real progress in such a complex field.

## How the Modules Are Structured

<img width="2079" height="783" alt="0-PT-Process" src="https://github.com/user-attachments/assets/ed591b8d-5330-42cb-9c1d-484bb9dbcfd0" />

The modules are arranged to guide you through the penetration testing process in a logical order. This applies whether you are a beginner or an experienced tester who feels stuck in certain areas.

Many tasks are designed to shape how you think, not just what commands you run. The goal is to help you develop analytical habits, learn to question assumptions, and recognise patterns. These skills cannot be taught in a single lesson. They are built gradually through repetition and hands-on practice.

You can think of this like learning a musical instrument. You can memorise theory and terminology, but without regular practice, you will not be able to perform. Penetration testing works the same way. Deep hands-on experience is what turns knowledge into capability.

## Pre-Engagement

The pre-engagement phase is where everything is defined before any testing begins. This is where scope, limitations, responsibilities, and expectations are documented. Contracts and rules of engagement are agreed upon, and both you and the client confirm what is and is not allowed.

<img width="2098" height="788" alt="0-PT-Process-PRE" src="https://github.com/user-attachments/assets/1564b131-8369-4b7b-9358-8839515e2e6d" />

From here, the process moves in only one direction.

### Foundation Modules Supporting This Stage

1. Learning Process

2. Linux Fundamentals

3. Windows Fundamentals

To understand how systems communicate, you must also understand networking concepts and terminology:

4. Introduction to Networking

Web applications are a category of their own. Before you attack them, you must understand how they work behind the scenes:

5. Introduction to Web Applications

6. Web Requests

7. JavaScript Deobfuscation

Modern environments also rely heavily on centralised management:

8. Introduction to Active Directory

9. Getting Started

## Information Gathering

Information gathering is the backbone of every penetration test. Every decision you make later depends on what you learn here.

Your goal is to identify targets, understand their role, and build a mental map of the environment. Rushing this phase almost always leads to wasted time and missed opportunities later. Good organisation, patience, and detailed note-taking are critical.

<img width="2120" height="787" alt="0-PT-Process-IG" src="https://github.com/user-attachments/assets/cbcf3026-d994-4caf-a4be-5a93660d3ae9" />

From here, you naturally move into vulnerability assessment.

### Modules Focused on This Stage

10. Network Enumeration with Nmap

11. Footprinting

12. Information Gathering - Web Edition

13. OSINT: Corporate Recon

## Vulnerability Assessment

This stage has two sides. One involves identifying known vulnerabilities using automated tools. The other involves manual analysis and creative thinking.

Automated tools tell you what is known. Analysis helps you find what is possible. This is where you connect information, understand workflows, and identify unintended behaviour. The stronger your technical background, the better you will be at this.

<img width="2108" height="808" alt="0-PT-Process-VA" src="https://github.com/user-attachments/assets/44c0f2e2-ea2a-441c-8bd8-43f4078c155b" />

At this point, you may move in several directions depending on what you find:

- Exploitation, if you are ready to attempt initial access
- Post-exploitation, if you already have access
- Lateral movement, if the environment allows it
- Back to information gathering, if gaps remain

### Relevant modules here include:

14. Vulnerability Assessment

15. File Transfers

16. Shells & Payloads

17. Using the Metasploit-Framework

## Exploitation

Exploitation is where you actively attempt to abuse weaknesses identified earlier. This stage relies heavily on everything you have already learned.

Different organisations may use the same software but configure it in completely different ways. Your job is to adapt. Exploitation is rarely a single step; it is a process of testing assumptions and adjusting based on results.

<img width="2100" height="810" alt="0-PT-Process-EX" src="https://github.com/user-attachments/assets/4191ce5d-41e9-44ca-8376-fda34ef8f643" />

From here, you may move into:

- Local information gathering
- Privilege escalation
- Lateral movement
- Proof-of-concept creation

### Relevant modules here include:

This part of the path focuses on network services, exposed protocols, user behaviour, and Active Directory environments:

18. Password Attacks

19. Attacking Common Services

20. Pivoting, Tunneling & Port Forwarding

21. Active Directory Enumeration & Attacks

## Web Exploitation

Web applications deserve special treatment because of their complexity and prevalence. They often represent the largest attack surface, especially during external assessments.

Web exploitation requires strong enumeration skills and an understanding of how multiple technologies interact, including databases, frameworks, and backend logic.

### Modules in this area include:

22. Using Web Proxies

23. Attacking Web Applications with Ffuf

24. Login Brute Forcing

25. SQL Injection Fundamentals

26. SQLMap Essentials

27. Cross-Site Scripting (XSS)

28. File Inclusion

29. Command Injections

30. Web Attacks

31. Attacking Common Applications

## Post-Exploitation

Once you gain access, the next challenge is understanding what you can do from inside the system. Services are often deliberately restricted, and bypassing those restrictions requires deep knowledge of the operating system.

This stage involves internal enumeration, privilege escalation, and preparation for further movement through the network. You must be able to navigate different versions of Windows and Linux confidently and recognise common weaknesses.

<img width="2162" height="799" alt="0-PT-Process-POEX" src="https://github.com/user-attachments/assets/56ae4c55-0aef-4d31-8e97-3e724265d0ed" />

### Modules here include:

32. Linux Privilege Escalation

33. Windows Privilege Escalation

## Lateral Movement

Lateral movement allows you to expand your access across the network. This does not always require administrator privileges, but it does require a solid understanding of authentication, trust relationships, and network layout.

<img width="2132" height="785" alt="image" src="https://github.com/user-attachments/assets/05b26250-cb0c-4a2c-9db8-85eccee5bd50" />

This stage often loops back into vulnerability assessment and pillaging on newly accessed systems, reinforcing the iterative nature of real penetration tests.

## Proof-of-Concept

A proof-of-concept exists to demonstrate that a vulnerability is real and reproducible. Clients will use it to validate findings before making changes that could affect critical systems.

A good PoC is clear, concise, and safe. It supports remediation without causing unnecessary risk or disruption.

<img width="2108" height="803" alt="0-PT-Process-POC" src="https://github.com/user-attachments/assets/20f8e9a8-429d-4bce-b23f-bc825d0ce58b" />

> At this stage, automation may become useful.

### Relevant modules here include:

34. Introduction to Python 3

## Post-Engagement

The final stage focuses on clean-up and reporting. Anything you uploaded, modified, or created during testing must be removed. Your documentation should clearly explain what was done, what was found, and how the client can verify your actions internally.

> Accurate, well-structured reporting is just as important as technical skill.

### Relevant modules here include:

35. Documentation & Reporting

36. Attacking Enterprise Networks
