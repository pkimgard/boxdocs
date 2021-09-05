*2021-09-02*

*kimm3*

# Walkthrough: Overpass 2 - Hacked
Platform: TryHackMe

Difficulty: Easy

- [Link](https://tryhackme.com/room/overpass2hacked)

Recommended for this box:
- Hackthebox Academy, Introduction to network analysis

## Setup
![ll](assets/markdown-img-paste-20210901111254535.png)

## Analysis
![statis-init](assets/markdown-img-paste-20210901111355354.png)

![conversations](assets/markdown-img-paste-20210901111543497.png)

- We have a 5 minute dump and almost 4000 packages captured.
- Most conersations are between '170.45' and '170.59'.
- Port 80, 2222 and 4242(<-My favourite).
- '170.45 is probably our machine being hacked.'

I'll start with the honeypot port 4242, because reasons.

Wireshark filter: `tcp.port == 4242`

There's one stream, right click any package and choose follow -> TCP stream.

(Filter: `tcp.stream eq 3`)

![4242-1](assets/markdown-img-paste-2021090111252866.png)

This shows the attacker gaining acceess as 'www-data', which means we should check out the http data next to find the foothold he used, there's a 'payload.php' file so we can search for that string later.

He then continues to check sudo privileges for user 'james', which he has a password for. 'james' unfortunately has full sudo privs.

He checks the shadow file, probably copying the hashes for offline cracking, and then clones a git repository of a ssh-backdoor.

![ssh-backdoor](assets/markdown-img-paste-20210901113036850.png)

![backdoor-setup](assets/markdown-img-paste-20210901113145420.png)

He generates RSA keys with no passphrase and starts the server, might be the conversation at port 2222 that we saw?

So far we have these credentials:

![creds](assets/markdown-img-paste-20210901115216308.png)

Let's try to crack the other hashes in the shadow file.

```
curl "https://raw.githubusercontent.com/drtychai/wordlists/master/fasttrack.txt" > tmp/fasttrack.txt
cat creds/pass.txt tmp/fasttrack.txt > tmp/wl.txt
tail -n 5 creds/shadow | cut -d ":" -f 2 > creds/hash-shadow
hashcat -a 0 -m 1800 creds/hash-shadow tmp/wl.txt
```

![cracked](assets/markdown-img-paste-2021090111503483.png)

Well that's nice, all of them cracked in one sec.

So the obvious way of getting back in is exploiting the ssh backdoor, but first let's find out how they delivered the 'payload.php'.
### Going back to http
There is not that much http traffic, if it were I would filter it out with `http.request.method == POST` to see all POST requests. There's only one in the pcap-file.

We can easily export any captured http files under "File -> Export objects -> HTTP".

![export-http](assets/markdown-img-paste-20210901115945851.png)

![payload.php](assets/markdown-img-paste-20210901120117224.png)

A one-liner named pipe/sh/nc revshell, in php.

![developmentform](assets/markdown-img-paste-20210901120632496.png)

They uploaded it through the development page that has a upload form.

![uploadspage](assets/markdown-img-paste-20210901120340519.png)

And then probably triggered with http(curl or browser), to '[address]/uploads/payload.php'.

![request-payload](assets/markdown-img-paste-20210901120530472.png)

Let's get back in.
## Geting access
Looking at the ssh-backdoor code, it has a defualt hash:

![salt](assets/markdown-img-paste-20210901121653684.png)

And a hardcoded salt:

![pwhandler](assets/markdown-img-paste-20210901121705180.png)

![pwhash](assets/markdown-img-paste-20210901121826130.png)

To verify the password, it gets hashed(SHA512) with the salt. Let's try to crack that.

![hc-wiki](assets/markdown-img-paste-2021090112232357.png)

```
echo -n "1c362db832f3f864c8c2fe05f2002a05" >> creds/hashbackdoor
tr "\n" ":" < creds/hashbackdoor > creds/hashsaltbackdoor
hashcat -a 0 -m 1710 creds/hashsaltbackdoor tmp/rockyou.txt
```

![hashcracked](assets/markdown-img-paste-20210901140456515.png)

Looking at port 2222 and following a stream in the capfile, we can confirm that port 2222 was used for SSH.

![ssh](assets/markdown-img-paste-2021090114070233.png)

![usertxt](assets/markdown-img-paste-20210901082009388.png)

Seems that they changed all the passwords, so we can't sudo as they did.

Interesting file in james home folder, '.suid-bash', which has setuid set.

![setuidbash](assets/markdown-img-paste-20210901082955556.png)

Seems to be a bash binary.

![bashbinary](assets/markdown-img-paste-20210901083037646.png)

If we run it we get a bash prompt, but still with user james.

![bashuserjames](assets/markdown-img-paste-20210901084434439.png)

Had to google for this and the awnser is in the man page for bash.

![poption bash](assets/markdown-img-paste-20210901085052495.png)

Basically if the the user(james in this case) and effective user(root) does not match, root becomes effective user with the p option.

![roottxt](assets/markdown-img-paste-20210901085722839.png)

![indexp80](assets/markdown-img-paste-20210901081754829.png)
