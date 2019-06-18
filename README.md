# purple-blossom
This is where I keep box writeups from boxes I've cracked, usually vulnhub.

[Basic Pentest 1](#writeup-of-basic-pentest-1-from-vulnhub)


# Writeup of Basic Pentest 1 from [VulnHub.](https://www.vulnhub.com/entry/basic-pentesting-1,216/)
Before I get into it, here's the description of the box from *Josiah Pierce*, the box's author:

Name: Basic Pentesting: 1

Date release: 8 Dec 2017

Author: Josiah Pierce

Series: Basic Pentesting

desc:

This is a small boot2root VM I created for my university’s cyber security group. It contains multiple remote vulnerabilities and multiple privilege escalation vectors. I did all of my testing for this VM on VirtualBox, so that’s the recommended platform. I have been informed that it also works with VMware, but I haven’t tested this personally.
This VM is specifically intended for newcomers to penetration testing. If you’re a beginner, you should hopefully find the difficulty of the VM to be just right.
Your goal is to remotely attack the VM and gain root privileges. Once you’ve finished, try to find other vectors you might have missed! If you enjoyed the VM or have questions, feel free to contact me at: josiah@vt.edu
If you finished the VM, please also consider posting a writeup! Writeups help you internalize what you worked on and help anyone else who might be struggling or wants to see someone else’s process. I look forward to reading them!

## Tools used: 
All the tools I ended up using are available in [Demon Linux](https://www.demonlinux.com/about.php) developed by [Weaknet  Labs.](https://www.weaknetlabs.com/) It's just personal preference, any pentesting VM should suit the needs of this VM.

## Attack vectors
I ended up finding three. I'm sure there's more, but here's the ones I found:

## FTP
After booting up both machines, I ran `netdiscover` to get the IP of the target machine, which was **192.168.48.130**.

From there, I ran `nmap -AT4 192.168.48.130` to get some details about the system.
*If anyone is unfamiliar with Nmap args, -A is os detection, version detection, script scanning, and traceroute, whereas T4 prohibits the dynamic scan delay from exceeding 10 ms* 

![The output of the Nmap scan.](https://cdn.discordapp.com/attachments/590366497797570561/590367347752173583/unknown.png)


This shows us that there's 3 open ports: 21, 22, and 80. The services running on these ports are (in order): 
* ProFTPD 1.3.3c
* OpenSSH 7.2p2
* Apache 2.4.18

I figured that these ports would probably only be open if there were vulnerabilities on them, so I began searching with the first one.
I immediately searched the ProFTPD 1.3.3c service on [Exploit DB](https://www.exploit-db.com) and saw that there was a remote code execution exploit on that version of ProFTPD.

![The Exploit DB page for ProFTPD1.3.3c](https://cdn.discordapp.com/attachments/590366497797570561/590368754224070657/unknown.png)

And here's that exploit in Metasploit Framework.

![The search results in MSF](https://cdn.discordapp.com/attachments/590366497797570561/590370962424201232/unknown.png)

I set the RHOSTS to the IP of the box, and run the exploit.

![The exploit being run](https://cdn.discordapp.com/attachments/590366497797570561/590373659827372092/unknown.png)

A quick `ls` confirms this.

![Output of the ls command](https://cdn.discordapp.com/attachments/590366497797570561/590374839311794178/unknown.png)

I `cat /etc/passwd` and `whoami` to confirm that I'm now the root user. Yay!

![Root!](https://cdn.discordapp.com/attachments/590366497797570561/590375350693789696/unknown.png)

## Web Server

From the Nmap scan, we know that there's an Apache webserver running on port 80. I use OWASP ZAP's forced browse to enumerate subdirectories:

![Output of forced browse](https://cdn.discordapp.com/attachments/590366497797570561/590375778013675531/unknown.png)

And there's an interesting subdirectory called *secret.*

It appears to be a Wordpress site, as seen here:
![Wordpress site](https://cdn.discordapp.com/attachments/590366497797570561/590376790669787136/unknown.png)

From there I run `wpscan --url 192.168.48.130/secret --enumerate u --wp-content-dir wp-content`

Which shows that not only are there 19 vulnerabilites, but that *admin* is a valid user.

![WPScan output part 1](https://cdn.discordapp.com/attachments/590366497797570561/590377329205837835/unknown.png)
![WPScan output part 2](https://cdn.discordapp.com/attachments/590366497797570561/590377378447097857/unknown.png)

I once again use WPScan to attempt to crack the admin user's password:

![WPScan password cracking](https://cdn.discordapp.com/attachments/560884796029403139/590377958552633374/unknown.png)

And we see a valid user:pass combo, admin:admin.

![WPScan password output](https://cdn.discordapp.com/attachments/590366497797570561/590378132352008223/unknown.png)

I use the `unix/webapp/wp_admin_shell_upload` exploit in MSF to gain a Meterpreter shell.

![Meterpreter Shell!](https://cdn.discordapp.com/attachments/590366497797570561/590379009943273517/unknown.png)

I run the Meterpreter command `shell` and then test it with `cat /etc/passwd`.

![Shell command](https://cdn.discordapp.com/attachments/590366497797570561/590379382783344652/unknown.png)

We're on a limited user account called www-data, as seen here:

![Limited acc](https://cdn.discordapp.com/attachments/590366497797570561/590380037270667275/unknown.png)

I try `ls -lah /etc/password` to see if I'm able to write to it and change the root password:

![Passwd file permissions](https://cdn.discordapp.com/attachments/590366497797570561/590380463445508134/unknown.png)

Darn. That'd be too easy. 

Next I downloaded the passwd and shadow files and tried using `unshadow passwd- shadow- >out.txt` to unshadow the pasword file and tried to crack it, but ran into no success.

From here I run `python -c ‘import pty; pty.spawn(“/bin/sh”)’` to get a shell, and then use `su - marlinspike` (the default user we found earlier while using `cat /etc/passwd` with the password marlinspike. And it works!

![Marlinspike](https://cdn.discordapp.com/attachments/590366497797570561/590381443793027092/unknown.png)

And then I run sudo su with the same password to get root.

![Root!](https://cdn.discordapp.com/attachments/590366497797570561/590381887550259200/unknown.png)

## SSH

Lastly, I log into SSH just using the same credentials we already discovered. Not really a hack, but eh.

![SSH](https://cdn.discordapp.com/attachments/590366497797570561/590382284826214430/unknown.png)

## Closing

And thats's it for this box!! It was really fun :)
