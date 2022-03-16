> sqli, password cracking,abusing sudo

# Ports

* **80** : Apache httpd 2.4.6
* **22** : OpenSSH 7.4
* **3306** : mariadb

### Port 80, 22 are used for exploiting

# 80
1. robots.txt contains some entries.Only /administrator can be accessed.
2. **Joomla!** CMS is running on the server (from \<meta> tag in the \<head> of the page)
3. Joomla version is determined by navigating to /language/en-GB/en-GB.xml.  Joomla! version is 3.7.0
4. This version of Joomla! is vulnerable to sqli (CVE-2017-8917)
5. Joomla! administration is at /administrator which requires valid credentials. Default credentials didn't work.

# 22
1. OpenSSH version 7.4
2. This version of openssh is vulnerable to a user enumeration vulnerability. CVE-2018-15473

# 3306
1. Sql server is running on this port, which isn't accessible from outside.


# EXPLOITATION

To exploit the sqli vulnerability in Joomla! we could use sqlmap or the metasploit module for this vulnerability. But neither of them seemed to work with this installation of the CMS.
[This](https://github.com/stefanlucas/Exploit-Joomla) exploit written in python from github for this vulnerability worked for me in this case. The exploit works with python2 and has compatibility issues with python3

`python joom.py http://10.10.132.221` 
After the exploit is finished we get output as `['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']`  from the table `fb9j5_users`

We have username as 'jonah' and a **salted bcrypt hash** of jonah's password. We can use John the ripper to crack this hash.

## John The Ripper
1. `john hash.txt --format=bcrypt --wordlist=~/Projects/SecLists/Passwords/Common-Credentials/100k-most-used-passwords-NCSC.txt`
2. We obtain password as `spiderman123`. Note that we can also use rockyou.txt.

After logging into /administrator using the credentials `jonah` and `spiderman` we can easily spawn a reverse shell by editing the template file as explained [here](https://vk9-sec.com/reverse-shell-on-any-cms/).
In this case to make things easier we directly edit the **index.php** of the `Beez3 Details and Files`  template and replace its content with the [php reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) and configure its port and ip to ours. After that we click on save and then hit 'template  preview' to get the connection to our listener. 

# Actions in the low privilege shell
1. We get the reverse shell as `apache` user who doesn't have much permissions.
2. In the /home directory home folder for the `jjameson` user exists.
3. Privilege escalation checklist has been done and nothing interesting came up. We navigate to our website directory in /var/www/html and look through its contents to see a 'configuration.php' file which might contain useful info.
4. In the 'configuration.php' there was a password for the sqldb as `nv5uz9r3ZEDzVjNu`. We can try this password for root , jjameson as their is a chance that they may reuse their system password for other services too.
5. We ssh as jjameson with password `nv5uz9r3ZEDzVjNu`. Logged in successfully
6. `sudo -l` shows that we have NOPASWD permission for `yum` the package manager. 
7. [GTFOBins](gtfobins.github.io/) has an entry for exploiting sudo permission of `yum`. We are going with the second method in the website that spawns an interactive root shell by loading a custom plugin.

We are now root.
