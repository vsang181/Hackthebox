Pilgrimage
Linux . Easy
Enumeration
Initial Nmap Scan
Our first step in probing the web application was an initial Nmap scan using the following command:
nmap -sCV -p- {machine IP}
This comprehensive scan returned a domain name for our target IP, which was subsequently added to the /etc/hosts file to enhance readability in further exploration.
Exploring the Web Application
Following the initial scan, we began investigating the web application's features. This exploration revealed:
A registration page, which automatically logs in new users
A manual login page
A home page featuring an image shrinking tool which allows users to upload images, subsequently displaying the resized image.
Once logged in, we were directed to a 'Dashboard' page. This interface displayed the original image file name and a link to the shrunken image, which had been assigned a new name by the application.
Detailed Nmap Scan
To gain further insight into the underlying infrastructure, a second, more comprehensive Nmap scan was conducted. This scan revealed a hidden .git directory, which offered a potential opportunity for further investigation.
Extracting Hidden Git Directory
In order to extract the contents of the hidden .git directory, we employed the git-dumper tool. This allowed us to gain access to the web application's source code, providing valuable information about the application's structure and functionality.
The source code comprised several different files for login, logout, register, and dashboard functionality, as well as an ELF 64-bit LSB executable called 'magick'. This command-line program is used to resize the images uploaded by users.
Source Code Analysis
Detailed scrutiny of the index.php file revealed a PHP script that facilitates image uploads, resizing, and storage. The script initializes a session, imports a package named "bulletproof" to handle image uploads, checks user authentication, processes image uploads, and renders HTML for the webpage.
We also discovered a SQLite database path mentioned within the dashboard.php file.
Investigating ImageMagick
In order to learn more about the potential vulnerabilities of the 'magick' tool, we downloaded it to our local system and checked the version.
Following a quick Google search, we discovered an Arbitrary File Read vulnerability in this version of ImageMagick. The provided Proof-of-Concept (POC) indicated that this vulnerability could be exploited for further investigations.
