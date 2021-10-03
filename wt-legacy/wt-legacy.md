*2021-10-02*

*kimm3*

# Walkthrough: Legacy
Platform: HackTheBox

Difficulty: Easy

- [Link](https://app.hackthebox.eu/machines/2)

This box is on TJ Null/NetSec Focus list for OSCP-like machines.
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.10.4; ping $IP
```
## Scans and enumeration
`sudo nmap $IP -p- -Pn -A -v -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-20211002110608190.png)

Windows XP, SMB and possible RDP:

`smbclient -N -L $IP`

![smb-timeout](assets/markdown-img-paste-20211003113902670.png)

Checking SMB version with msf:

![smb-version](assets/markdown-img-paste-20211003110829377.png)

If it's SMBv1, it might be vulnerable to ms17-010(eternalblue).

![ms17-010-check](assets/markdown-img-paste-2021100311252667.png)

Let's try.
## Exploit
I used another exploit than on the 'blue' machine because it had payloads available that was not 64-bit, and this is Windows XP - so most likely 32-bit.

![merp](assets/markdown-img-paste-20211003112818255.png)

Works and we got the System account, so we can retrieve both flags.

![usertxt](assets/markdown-img-paste-20211003113136703.png)

![roottxt](assets/markdown-img-paste-20211003113030629.png)

## Alternative
Nmap is pretty quick and easy to enumerate vulnerabilities, there's one more possible exploit to explore if you like.

![nmap-vuln](assets/markdown-img-paste-20211003113425961.png)
