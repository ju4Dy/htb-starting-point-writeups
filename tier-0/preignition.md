Yooo guys, it's ju4dy the landlord again.

# Preignition — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 0 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.57.168` |
| **Skills** | Web enumeration, directory brute forcing, default credentials |

## Overview

Preignition is a very easy Linux host running a single nginx web server. The default nginx landing page hides
an administrative login that is not linked from anywhere, and directory brute forcing finds it. The login
accepts `admin:admin`.

The box is a short lesson with two parts: an application endpoint is only as hidden as its name, and a default
credential pair is a back door left open on purpose.

## Enumeration

A default TCP scan with service and version detection is enough to characterize the host.

```bash
nmap -sVC 10.129.57.168
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 09:45 +0100
Nmap scan report for 10.129.57.168
Host is up (0.54s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome to nginx!

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.37 seconds
```

Only port `80` is open, and the title `Welcome to nginx!` is the stock landing page shipped with the package.
That page says nothing about the application, so the next step is to look for content that is not linked from
the index.

### Directory brute forcing

Gobuster is pointed at the site with the small `directory-list-2.3` wordlist and PHP extensions appended to
every entry:

```bash
gobuster dir -u http://10.129.57.168/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt -x php
```

```text
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.57.168/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/admin.php            (Status: 200) [Size: 999]
```

The scan was stopped early, having covered only about two percent of the wordlist, but the hit arrived before
that: `/admin.php` returns `200` with a 999-byte response. A directory-only wordlist would have missed it,
which is why `-x php` matters when the server is known to run PHP-backed content.

Extending the same run with a broader extension set and a larger wordlist is the natural follow-up:

```bash
gobuster dir -u http://10.129.57.168/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt
```

## Exploitation — default credentials

Browsing to `http://10.129.57.168/admin.php` presents a small login form. The credentials `admin:admin` are
accepted on the first attempt, which identifies the weakness as a default installation credential rather than
a flaw in the code.

<!-- screenshot: admin.php login form -->

## Capturing the flag

Once authenticated, the admin panel returns the flag for the box.

<!-- screenshot: flag displayed in the admin panel after login -->

## Key takeaways

- **Change default credentials before an application is reachable.** `admin:admin` is the first pair any
  tester tries, and it is also the first pair automated scanners try.
- **A stock web server page is not a dead end.** The nginx landing page concealed an entire application; the
  absence of links on the index page is a reason to enumerate, not a reason to stop.
- **Wordlist choice and extension handling decide the result.** `-x php` surfaced the endpoint, and the same
  scan with a directory-only wordlist would have reported nothing.
- **Do not rely on an obscure URL as access control.** `/admin.php` was found in seconds by brute forcing;
  authentication and authorization have to carry the security, not the path.
- **Detection should treat broad 404 sweeps as hostile.** Rate limiting and alerting on a high volume of
  requests from a single source would make this kind of enumeration noisy and slow.
