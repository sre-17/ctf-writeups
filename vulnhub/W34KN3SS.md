> Virutal host, ssh

# Ports

* **80, 443** : Default apache setup page
* **22** : SSH
### Port 80, 22 are used for exploiting

# 80
1. Gobuster gave pages /uploads, /upload.php, /blog. Didn't find anything useful
2. Nmap scan of the target gave infomation about a `weakness.jt` in the ssl info
3. Added `weakness.jt` to /etc/hosts.
4. `weakness.jt` page contains an ascii rabbit with n30 written below. Possibly a user name `n30` in the system.
5. Gobuster of weakness.jt gave /private
6. /private page contains an ssh public key along with a note that openssh 0.9.8c was used to create the key 

# EXPLOITATION

Openssl 0.9.8c contains a vulnerability that causes it to produce only 65536 types of sshkeys. [exploitdb page](https://www.exploit-db.com/exploits/5622)

We can use a list of private-public keys to determine the private key for the given public key. List is obtained from [this github repo.](https://github.com/offensive-security/exploitdb-bin-sploits/raw/master/bin-sploits/5622.tar.bz2 (debian_ssh_rsa_2048_x86.tar.bz2))

## grep
1. `grep -l -r "AAAAB3NzaC1yc2EAAAABIwAAAQEApC39uhie9gZahjiiMo+k8DOqKLujcZMN1bESzSLT8H5jRGj8n1FFqjJw27Nu5JYTI73Szhg/uoeMOfECHNzGj7GtoMqwh38clgVjQ7Qzb47" .`
"Checking a part of the public key inside the folder of the github download"

2. `./4161de56829de2fe64b9055711f531c1-2537.pub` is the result. Here `4161de56829de2fe64b9055711f531c1-2537` is the private key. We can use this to login without needing a password.

3. ssh private key file needs to have permissions set 600 or rw-------. So we set `chmod 600 ./4161de56829de2fe64b9055711f531c1-2537.pub`

Finally `ssh -i 4161de56829de2fe64b9055711f531c1-2537 -l n30`  n30 from the rabbit ascii


# Actions in the low privilege shell
1. `ls -la`
Shows a binary named `code` in the home folder of n30.
2. `file code`
Gives the file type as python byte code
3. `python code` runs the binary and gives a hash for the user. Which is unique for each run.
4. So we send it to our attacking machine to decompile it. 
5. In our machine we rename the file to code.pyc since the decompiler shows error otherwise
`uncompyle6 code.pyc`
6. The python code shows `username:password currenttime` used to create a sha256 hash in that format. So we get the password from the code as `dMASDNB!!#B!#!#33` 
7. We run `sudo -l` and enter the password to find that this user can do all actions as sudo
8. So we `sudo bash` to switch to the root .
9. `whoami` returns root
