# Exfiltrated
>**Platform:** OffSec Proving Grounds Practice  
>**OS:** Linux  
>**Difficulty:** Fundamental  
>**IP:** `192.168.162.163`  
>**Status:** Rooted

## Overview

## Attack Path

# Recon
### Nmap
I started with a full TCP-scan with service and version detection
```bash
sudo nmap -Pn -p- -sVC --min-rate 10000 $target -oN nmap.txt
```
```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-04 23:25 -0400
Nmap scan report for 192.168.162.163
Host is up (0.071s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c1:99:4b:95:22:25:ed:0f:85:20:d3:63:b4:48:bb:cf (RSA)
|   256 0f:44:8b:ad:ad:95:b8:22:6a:f0:36:ac:19:d0:0e:f3 (ECDSA)
|_  256 32:e1:2a:6c:cc:7c:e6:3e:23:f4:80:8d:33:ce:9b:3a (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Did not follow redirect to http://exfiltrated.offsec/
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-robots.txt: 7 disallowed entries 
| /backup/ /cron/? /front/ /install/ /panel/ /tmp/ 
|_/updates/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.16 seconds
```
### Findings
| Port   | Service | Version                                                      |
| ------ | ------- | ------------------------------------------------------------ |
| 22/tcp | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0) |
| 80/tcp | HTTP    | Apache httpd 2.4.41 (Ubuntu)                                 |

Nmap scan revealed port 22 and 80 running on the target machine
And we see the scanner being redirected to a domain name. To proceed, we'll add exfiltrated.offsec to our local /etc/hosts and start enumerating the web application
#### port 80 - HTTP
There's an Blog website Powered by Subrion CMS running on port 80 
<img width="1180" height="634" alt="Screenshot 2026-10-05 at 9 10 10 AM" src="https://github.com/user-attachments/assets/c114ee5b-ddae-4f21-b2e4-6a865ac06bdf" />

http://exfiltrated.offsec/panel/
admin:admin 
<img width="651" height="414" alt="Screenshot 2026-10-05 at 12 01 38 PM" src="https://github.com/user-attachments/assets/8f63da35-4117-48a0-b232-b16d58878420" />

http://exfiltrated.offsec/uploads/php-reverse-shell.phar
<img width="841" height="200" alt="Screenshot 2026-10-05 at 12 02 18 PM" src="https://github.com/user-attachments/assets/2fb6afd3-f5ad-4643-ab23-8e6fdd8ef4a2" />

<img width="933" height="405" alt="Screenshot 2026-10-05 at 12 04 18 PM" src="https://github.com/user-attachments/assets/cc79b178-da57-4ca9-b589-a66163110d74" />



# Enumeration

# Exploitation
