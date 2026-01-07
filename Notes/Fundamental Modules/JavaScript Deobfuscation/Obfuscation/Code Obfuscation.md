# Code Obfuscation

Before learning about deobfuscation, it is necessary to understand code obfuscation. Without understanding how code is obfuscated, it may not be possible to successfully deobfuscate it, particularly when a custom obfuscation method is used.

# What is Obfuscation

Obfuscation is a technique used to make a script difficult for humans to read while allowing it to function identically from a technical perspective, although performance may be reduced. This process is typically performed automatically using an obfuscation tool, which takes source code as input and rewrites it into a form that is significantly harder to interpret, depending on the design of the tool.

For example, obfuscators often convert code into a dictionary containing all words and symbols used within the script, then reconstruct the original logic at runtime by referencing entries from that dictionary. The following illustrates a simple JavaScript script that has been obfuscated:
JavaScript deobfuscation module with options for fast decode and special characters, showing obfuscated code.

Many interpreted languages publish and execute code without compilation, such as Python, PHP, and JavaScript. While Python and PHP are typically executed on the server side and are therefore hidden from end users, JavaScript is commonly executed on the client side within the browser. As a result, JavaScript source code is delivered to the user in cleartext and executed locally, which is why obfuscation is frequently applied to JavaScript.

![obfuscation_example](https://github.com/user-attachments/assets/027d0f52-0453-4e92-a972-8587a8ec9414)

# Use Cases

There are several reasons why developers may choose to obfuscate their code. One common reason is to conceal the original implementation and functionality to prevent unauthorised reuse or copying, thereby increasing the difficulty of reverse engineering. Another reason is to add an additional layer of protection when handling authentication or encryption logic, reducing the likelihood of direct exploitation of vulnerabilities present in the code.

It must be noted that performing authentication or encryption on the client side is not recommended, as client-side code is inherently more exposed to attacks.

The most common use of obfuscation is for malicious purposes. Attackers frequently obfuscate malicious scripts to evade detection by intrusion detection and prevention systems. In the next section, a simple JavaScript script will be obfuscated and executed both before and after obfuscation to observe any differences in behaviour.
