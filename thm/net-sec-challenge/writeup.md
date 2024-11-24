# net-sec-challenge

[Room](https://tryhackme.com/r/room/netsecchallenge)

Let's go.

# What is the highest port number being open less than 10,000?

```bash
nmap -p 1-10000 $LHOST
Starting Nmap 7.80 ( https://nmap.org ) at 2024-11-24 01:55 -03
Nmap scan report for 10.10.86.16
Host is up (0.34s latency).
Not shown: 9995 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 213.44 seconds
```

# There is an open port outside the common 1000 ports; it is above 10,000. What is it?

Go with `nmap -p 10000-10100 $LHOST` to find it.

# What is the flag hidden in the HTTP server header?

If we go to http://LHOST, we'll find a Hello, World! message. Nothing interesting to see there.
Take a look at the response headers to find the flag.

# What is the flag hidden in the SSH server header?

There is a SMB share in port 445. What if we try with that? Let's use smbclient with
the no pass(-N) and the list(-L) flags.

```bash
smbclient -N -L //$LHOST
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
IPC$            IPC       IPC Serv</pre>
SMB1 disabled -- no workgroup available
```

But they don't help very much.

_header_ is the key here. Do `ssh $LHOST -v` and look for the flag.

# We have an FTP server listening on a nonstandard port. What is the version of the FTP server?

Use the port >10000 from above with nmap `nmap -sVC -p $PORT $LHOST` to get the answer.

# We learned two usernames using social engineering: eddie and quinn. What is the flag hidden in one of these two account files and accessible via FTP?

Time to use hydra. Remember that the FTP service isn't running on the standard port(21).

`hydra -L eddie -w -P rockyou.txt $LHOST ssh -s port` and `hydra -L quinn -w rockyou.txt $LHOST ssh -s $PORT`, will give us the users passwords.

Now, we can connect to the FTP server using `ftp eddie|jordan@$LHOST $PORT` and using the respective password as input.

# Browsing to http://$LHOST:8080 displays a small challenge that will give you a flag once you solve it. What is the flag?

Get to know the different types of scan of nmap.
