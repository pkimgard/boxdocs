*2021-10-29*

*kimm3*

# Walkthrough: Previse
Platform: HackTheBox

Difficulty: Easy

Good reference: [Ippsec, Bank](https://www.youtube.com/watch?v=JRPWFSzFaG0)

Uploaded after retirement:
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.11.104; ping $IP
```
## Scans and enumeration
`sudo nmap $IP -p- -Pn -v -A -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-20211023120231295.png)

![index](assets/markdown-img-paste-20211023120157778.png)

![loginfail](assets/markdown-img-paste-20211023120441101.png)

Tried som basic credentials but didn't work. Did som web fuzzing and found a couple of redirects, and this 'nav.php':

![navphp](assets/markdown-img-paste-20211023120744812.png)

If you look browse the page in Burp, you'll see that it lists a bunch of sites that we can't see. When browsing to one of the sites, it actually gets the site and then redirects you to the login page. It's a 302 FOUND with the actual content. Because of the 302, the browser will redirect you before you see it.

![target-list](assets/markdown-img-paste-20211029062907590.png)

Response when accessing 'accounts.php':

![response-accounts](assets/markdown-img-paste-20211029062726392.png)

In 'files.php' there's a link to a file.

![backupfile](assets/markdown-img-paste-20211029065753114.png)

We can easily replace the 302s with 200s and view them in browser with burp. Start the proxy, capture the responses and 'Match and Replace' 302 FOUND with 200 OK. Now you can surf via browser.

![intercept-response](assets/markdown-img-paste-20211029095202745.png)

![match-and-replace](assets/markdown-img-paste-20211029095216204.png)

From here, we can add a user under the accounts page and log in.

![account-created](assets/markdown-img-paste-20211029103840207.png)

There's one zip-file under files that contains the site.

![zip](assets/markdown-img-paste-20211029073653768.png)

![configphp](assets/markdown-img-paste-20211029073730488.png)

Saving these credentials for later.

![logsphp](assets/markdown-img-paste-20211029103148864.png)

In 'logs.php', a system call is made to run a python script. It takes a parameter via a POST request. We can intercept this request and change it to run system commands for us.

**Note**

The encoding is important, run the calls in burps repeater and turn on "URL-Encode as you type".

Test command:

On host: `sudo tcpdump -i tun0 icmp`

delim parameter:

![delim](assets/markdown-img-paste-20211029104221772.png)

result(4 icmp echo request/replies):

![tcpdump](assets/markdown-img-paste-20211029104205811.png)
## Foothold
Turn off the proxy, make sure you're logged in, and use repeater to send this command.

![delim-shell](assets/markdown-img-paste-20211029104543854.png)

![shell](assets/markdown-img-paste-2021102910482703.png)

## Lateral movement
There's a user in the home folder with the user.txt, m4lwhere. But we cannot read it.

Let's try to connect to the db with the credentials we found.

```
which mysql
mysql -u root -p -D previse
```

![sql-query](assets/markdown-img-paste-20211029105653345.png)

We get the table name from the 'accounts.php'.

`SELECT * FROM accounts;`

![accounts-table](assets/markdown-img-paste-20211029105824144.png)

![hashcat-examples](assets/markdown-img-paste-20211029111631129.png)

`hashcat -a 0 -m 500 m4lwherehash ../sl/Passwords/Leaked-Databases/rockyou.txt`

![cracked](assets/markdown-img-paste-2021102911294098.png)

![usertxt](assets/markdown-img-paste-20211029113105247.png)
## Privilege Escalation
![sudol](assets/markdown-img-paste-20211029113224377.png)

![backupsh](assets/markdown-img-paste-20211029113319498.png)

Looks promising.

![pathandgzip](assets/markdown-img-paste-20211029114024466.png)

Since the script dosen't execute gzip with absolute path, we can exploit it by simply changing the path.

```
cd # Go to a writable folder
echo "/bin/bash" > gzip # Create a file that launches a shell
chmod 777 gzip # Make it executable
export PATH=/home/m4lwhere/:$PATH # Change path so that our file comes first
sudo /opt/scripts/access_backup.sh # Execute script as root
```

![rootshell](assets/markdown-img-paste-2021102911525074.png)

The shell didn't return any output. Started another reverse shell and got the output from there.

![roottxt](assets/markdown-img-paste-2021102911514221.png)
