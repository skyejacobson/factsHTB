# factsHTB
Writeup of the seasonal Facts Hack the Box machine


Initial nmap scan of the machine reveals 2 PoA being a HTTP site and SSH service being hosted.
'''
┌──(root㉿kali-linux-2024-2)-[/]
└─# nmap -sV -sC 10.129.4.222
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-16 22:56 -0600
Nmap scan report for 10.129.4.222
Host is up (0.24s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)
80/tcp open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx/1.26.3 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.28 seconds
'''
