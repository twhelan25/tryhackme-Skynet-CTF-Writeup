
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
Let's check out the email samba password reset. It sure enough it has Mile's password for smb login.
Now let's use smbclient again to log onto the smb share:
```bash
smbclient -U milesdyson //$ip/milesdyson
```
And we're in! There's a lot of irrelivent files here, so let's just cd into notes. There's a file called important.txt and I bet it is important. Let's get it.

![important](https://github.com/user-attachments/assets/8fd28c32-991c-4256-a266-fce4f6941628)
