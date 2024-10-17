# Ninja Skills

At some point I just did

```
$ find / -type f -readable -print 2> /dev/null | grep -E "8V2L|bn
y0|c4ZX|D8B3|FHl1|oiMO|PFbD|rmfX|SRSq|uqyw|v2Vb|X1Uy"
/mnt/D8B3
/mnt/c4ZX
/var/FHl1
/var/log/uqyw
/opt/PFbD
/opt/oiMO
/media/rmfX
/etc/8V2L
/etc/ssh/SRSq
/home/v2Vb
/X1Uy
```

To find all readable files of the list. It might not be the best idea but it works.

## Which of the above files are owned by the best-group group?

Listing with `ls` reveals a `files` dir, but it's empty. So, what now? `ls -a` inside `files` doesn't show anything.
`history` tells that there is an `ec2-user` dir or I could have done `cd ..`.

```
$ ls -la
total 36
drwxr-xr-x  5 root       root        4096 Oct 23  2019 .
dr-xr-xr-x 25 root       root        4096 Oct 17 03:36 ..
drwx------  3 ec2-user   ec2-user    4096 Oct 23  2019 ec2-user
drwx------  2 newer-user newer-user  4096 Oct 23  2019 newer-user
drwx------  3 new-user   new-user    4096 Oct 23  2019 new-user
-rw-rw-r--  1 new-user   best-group 13545 Oct 23  2019 v2Vb
```

We can see that `v2Vb` is owned by `new-user` and that `best-group` owns it too. Also, there are
four users: `root`, `ec2-user`, `newer-user`, `new-user`.

But a simple answer is found with `find / -group best-group`, that shows that `/mnt/D8B3` and `/home/v2Vb` are owned
by best-group group.

## Which file's owner has an ID of 502?

A quick man read on find tell us that the uid flag can be used to locate files owned by the given user ID.
If we test it: `find / -uid 502` or with a redirect of _all_ errors to /dev/null

```
find / -uid 502 2> /dev/null
/var/spool/mail/newer-user
/home/newer-user
/X1Uy
```

And the file X1Uy is the answer, but as a not searched for answer we can also see that 502 is the ID of newer-user.

## Which of these files is executable by everyone?

This time let's start using patterns in grep: `find / -type f -executable -print 2> /dev/null | grep -E "8V2L|
bny0|c4ZX|D8B3|FHl1|oiMO|PFbD|rmfX|SRSq|uqyw|v2Vb|X1Uy"`. It's long to write but now we have an universal filter.
It outputs: `/etc/8V2L` which is the answer.

## Which of these files contain an IP address?

```
$ find / -type f -readable -print 2> /dev/null | grep -E "8V2L|bny
0|c4ZX|D8B3|FHl1|oiMO|PFbD|rmfX|SRSq|uqyw|v2Vb|X1Uy" | xargs -I'{}' grep -E "([0-9]{1,3}[\.])([0-9]{1,3}[\.])([0-9]{1,3}[\.])([0-9]{1,3}[\.])" '{}'
```

This command shows the contents of the file(with a 1.1.1.1 in the middle) but not the name of
such file.

```
find / -type f -readable -print 2> /dev/null | grep -E "8V2L|bny
0|c4ZX|D8B3|FHl1|oiMO|PFbD|rmfX|SRSq|uqyw|v2Vb|X1Uy" | xargs -I'{}' grep -HE "([0-9]{1,3})[\.]
([0-9]{1,3})[\.]([0-9]{1,3})[\.]([0-9]{1,3})" '{}'
/opt/oiMO:wNXbEERat4wE0w/O9Mn1.1.1.1VeiSLv47L4B2Mxy3M0XbCYVf9TSJeg905weaIk
```

The grep's `-H` flag makes it print the name of the file that matches the pattern.

## Which file has the SHA1 hash of 9d54da7584015647ba052173b84d45e8007eba94

```
$ sha1sum /mnt/c4ZX
9d54da7584015647ba052173b84d45e8007eba94  /mnt/c4ZX
```

## Which file contains 230 lines?

A command to find the lines of a file is `wc -l {file}`.

```
find / -type f -readable -print 2> /dev/null | grep -E "8V2L|bny
0|c4ZX|D8B3|FHl1|oiMO|PFbD|rmfX|SRSq|uqyw|v2Vb|X1Uy" | xargs -I'{}' wc -l '{}'
209 /mnt/D8B3
209 /mnt/c4ZX
209 /var/FHl1
209 /var/log/uqyw
209 /opt/PFbD
209 /opt/oiMO
209 /media/rmfX
209 /etc/8V2L
209 /etc/ssh/SRSq
209 /home/v2Vb
209 /X1Uy
```

Hence, the remaining file(that is not listed as readable by find) must be the answer: bny0.

## Bonus

Something that I've learned from this room: the `xargs` command, with it I can pass each
line from a multiline command output to another program without a for loop. For example:

```
find / -type f -readable -print 2> /dev/null | grep -E "8V2L|bny
0|c4ZX|D8B3|FHl1|oiMO|PFbD|rmfX|SRSq|uqyw|v2Vb|X1Uy" | xargs -I'{}' sha1sum '{}'
```

## References

- https://www.cyberciti.biz/faq/searching-multiple-words-string-using-grep/
- https://unix.stackexchange.com/questions/352020/how-to-pass-multiple-lines-to-a-parameter-without-a-for-loop
