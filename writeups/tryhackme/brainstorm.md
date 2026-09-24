# Brainstorm — TryHackMe

<p align="left">
  <img src="./assets/brainstorm/00-card.png" alt="Brainstorm machine card" width="650">
</p>

Reverse engineer a chat program and write a script to exploit a Windows machine.

## **Deploy Machine and Scan Network**

![brainstorm scan](./assets/brainstorm/brainstorm-scan.png)

## **Accessing Files**

![braintstorm ftp](./assets/brainstorm/braintstorm-ftp.png)

## **Access**

After enumeration, you now must have noticed that the service interacting on the strange port is some how related to the files you found! Is there anyway you can exploit that strange service to gain access to the system? 

It is worth using a Python script to try out different payloads to gain access! You can even use the files to locally try the exploit. 

If you've not done buffer overflows before, check [this](https://tryhackme.com/room/bof1) room out!

###### Answer the questions below

Read the description.

After testing for overflow, by entering a large number of characters, determine the EIP offset.  

Now you know that you can overflow a buffer and potentially control execution, you need to find a function where ASLR/DEP is not enabled. Why not check the DLL file.  

Since this would work, you can try generate some shellcode - use msfvenom to generate shellcode for windows.  

After gaining access, what is the content of the root.txt file?
5b1001de5a44eca47eee71e7942a8f8a

![brainstorm scan](./assets/brainstorm/brainstorm-scan.png)
![braintstorm ftp](./assets/brainstorm/braintstorm-ftp.png)


![fuzzing brainstorm](./assets/brainstorm/fuzzing-brainstorm.png)

![eip down](./assets/brainstorm/eip-down.png)
![offset 2012 brainstorm](./assets/brainstorm/offset-2012-brainstorm.png)
![badchar](./assets/brainstorm/badchar.png)
![jmp point](./assets/brainstorm/jmp-point.png)
![payload](./assets/brainstorm/payload.png)
![sending payload](./assets/brainstorm/sending-payload.png)
![app running](./assets/brainstorm/app-running.png)
![shell](./assets/brainstorm/shell.png)
![run chatserver get shell](./assets/brainstorm/run-chatserver-get-shell.png)
![new payload](./assets/brainstorm/new-payload.png)
![send buffer](./assets/brainstorm/send-buffer.png)
![shell machine](./assets/brainstorm/shell-machine.png)
![root flag](./assets/brainstorm/root-flag.png)
---

**Live version:** [read this write-up on my blog](https://debasjan.github.io/writeups/brainstorm/)
