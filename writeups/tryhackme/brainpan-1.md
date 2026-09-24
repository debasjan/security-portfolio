# Brainpan 1 — TryHackMe

<p align="left">
  <img src="./assets/brainpan-1/00-card.png" alt="Brainpan 1 machine card" width="650">
</p>

Reverse engineer a Windows executable, find a buffer overflow and exploit it on a Linux machine.


## **Deploy and compromise the machine**

Brainpan is perfect for OSCP practice and has been highly recommended to complete before the exam. Exploit a buffer overflow vulnerability by analyzing a Windows _exe_cutable on a Linux machine. If you get stuck on this machine, don't give up (or look at writeups), just try harder. 

  
All credit to [superkojiman](https://www.vulnhub.com/entry/brainpan-1,51/) - This machine is used here with the explicit permission of the creator <3

###### Answer the questions below

Deploy the machine.

Gain initial access

Escalate your privileges to root.

![nmap command](./assets/brainpan-1/nmap-command.png)
![nmap scan](./assets/brainpan-1/nmap-scan.png)
![port 9999 check](./assets/brainpan-1/port-9999-check.png)
![port 10000 check](./assets/brainpan-1/port-10000-check.png)
![gobuster port 10000](./assets/brainpan-1/gobuster-port-10000.png)
![brianpan exe](./assets/brainpan-1/brianpan-exe.png)
![brainpan export to win](./assets/brainpan-1/brainpan-export-to-win.png)
![fuzzing](./assets/brainpan-1/fuzzing.png)
![pattern create 900](./assets/brainpan-1/pattern-create-900.png)
![offset](./assets/brainpan-1/offset.png)
![byterray](./assets/brainpan-1/byterray.png)
![badchar unmodfied](./assets/brainpan-1/badchar-unmodfied.png)
![jump point](./assets/brainpan-1/jump-point.png)
![msfvenom payload](./assets/brainpan-1/msfvenom-payload.png)
![buffer script local](./assets/brainpan-1/buffer-script-local.png)
![msfvenom payload 2](./assets/brainpan-1/msfvenom-payload-2.png)
![payload script + shell machine](./assets/brainpan-1/payload-script-shell-machine.png)
![shell](./assets/brainpan-1/shell.png)
![sudo -l](./assets/brainpan-1/sudo-l.png)
![root](./assets/brainpan-1/root.png)
---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/brainpan-1/)
