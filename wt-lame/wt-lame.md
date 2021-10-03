*2021-09-29*

*kimm3*

# Walkthrough: Lame
Platform: HackTheBox

Difficulty: Easy

## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.10.3; ping $IP
```
## Scans and enumeration
`sudo nmap $IP -p- -Pn -A -v -oA scans/nmap-init`

![namp-init](assets/markdown-img-paste-20210929144344450.png)

![ftp-dir](assets/markdown-img-paste-20210929144438337.png)

![smbclient](assets/markdown-img-paste-20210929144525963.png)

Nothing on ftp or smb. Researching for distcc.

![distcc-google](assets/markdown-img-paste-20210929145026539.png)

`sudo nmap $IP -p 3632 --script vuln -oA scans/nmap-distcc-vuln`

![nmap-vuln](assets/markdown-img-paste-20210929144928664.png)

![searchsploit](assets/markdown-img-paste-20210929145145604.png)

It's vulnerable and we have a msf module for it.
## Foothold/Exploit
![shell](assets/markdown-img-paste-20210929145435628.png)

Running it gives us a shell and user.txt.

![usertxt](assets/markdown-img-paste-20210929145557919.png)


## Privilege Escalation
Doing basic enumeration.

Searched for setUIDs, nmap had it set and we can run shell commands from inside the interactive mode.

![setuidsearch](assets/markdown-img-paste-2021092915011615.png)

![roottxt](assets/markdown-img-paste-20210929150226122.png)
