# Daily Bugle — TryHackMe

<p align="left">
  <img src="./assets/daily-bugle/00-card.png" alt="Daily Bugle machine card" width="650">
</p>

Compromise a Joomla CMS account via SQLi, practise cracking hashes and escalate your privileges by taking advantage of yum.

## **Deploy**

![daily bugle](./assets/daily-bugle/daily-bugle.png)

###### Answer the questions below

Access the web server, who robbed the bank?
Spiderman

![scan](./assets/daily-bugle/scan.png)

![port 80 web](./assets/daily-bugle/port-80-web.png)


## **Obtain user and root**

![obtain user and root](./assets/daily-bugle/obtain-user-and-root.png)
Hack into the machine and obtain the root user's credentials.

###### Answer the questions below

What is the Joomla version?
3.7.0

*Instead of using SQLMap, why not use a python script!*  

What is Jonah's cracked password?
spiderman123

What is the user flag?
27a260fe3cba712cfdedb1c86d80442e

What is the root flag?
eec3d53292b1821868266858d7fa6f79

![gobuster common](./assets/daily-bugle/gobuster-common.png)
![gobuster medium](./assets/daily-bugle/gobuster-medium.png)
![joomla](./assets/daily-bugle/joomla.png)
![robots](./assets/daily-bugle/robots.png)
```
cmseek -u http://(target)
```
![cmseek](./assets/daily-bugle/cmseek.png)
![searchsploit joomla](./assets/daily-bugle/searchsploit-joomla.png)

![joomla exploit](./assets/daily-bugle/joomla-exploit.png)
![joomblah](./assets/daily-bugle/joomblah.png)
![joomblah 2](./assets/daily-bugle/joomblah-2.png)
![joomblah hash](./assets/daily-bugle/joomblah-hash.png)
![cat hash](./assets/daily-bugle/cat-hash.png)
![hahsid](./assets/daily-bugle/hahsid.png)
![bcrypt](./assets/daily-bugle/bcrypt.png)
![john cracked](./assets/daily-bugle/john-cracked.png)
![joomla jonah](./assets/daily-bugle/joomla-jonah.png)
![templates](./assets/daily-bugle/templates.png)
![templates new file](./assets/daily-bugle/templates-new-file.png)
![create php](./assets/daily-bugle/create-php.png)
![reverse shell](./assets/daily-bugle/reverse-shell.png)
![shell](./assets/daily-bugle/shell.png)
![shell2](./assets/daily-bugle/shell2.png)
![shell user](./assets/daily-bugle/shell-user.png)

![upload linpeas](./assets/daily-bugle/upload-linpeas.png)
![linpeas chmod](./assets/daily-bugle/linpeas-chmod.png)
![password](./assets/daily-bugle/password.png)
![password configuration](./assets/daily-bugle/password-configuration.png)
![jjameson](./assets/daily-bugle/jjameson.png)
![jjameson user flag](./assets/daily-bugle/jjameson-user-flag.png)
![jjameson linpeas](./assets/daily-bugle/jjameson-linpeas.png)
![bin yum](./assets/daily-bugle/bin-yum.png)
![gtfo yum](./assets/daily-bugle/gtfo-yum.png)
![root shell](./assets/daily-bugle/root-shell.png)
![root flag](./assets/daily-bugle/root-flag.png)
---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/daily-bugle/)
