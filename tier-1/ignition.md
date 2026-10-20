Yooo guys, it's ju4dy the landlord again.

# Ignition — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 1 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.1.27` |
| **Skills** | Virtual host discovery, content discovery, default credentials |

## Overview

Ignition is a Linux machine running nginx 1.14.2 with a single virtual host, `ignition.htb`. The interesting
part of the box is not an exploit but the credential on the administrative panel: the application is still
using the vendor default, `admin` / `qwerty123`, and that is enough to reach the flag.

It is a short box, but it is a useful reminder of two habits — follow the redirect the web server hands you,
and always try the default login before reaching for a wordlist.

## Enumeration

```bash
nmap -sVC 10.129.1.27
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 23:00 +0100
Nmap scan report for 10.129.1.27
Host is up (0.87s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Did not follow redirect to http://ignition.htb/
```

Only port `80` is open, served by nginx. The `http-title` line is the important one: the script reports
*Did not follow redirect to http://ignition.htb/*, which means the bare IP returns a redirect pointing at a
hostname. Requesting the site by IP therefore never shows the real application.

### Virtual host

The hostname from the redirect is added to the local hosts file so the browser and command-line tools resolve
it to the target:

```bash
echo "10.129.1.27 ignition.htb" | sudo tee -a /etc/hosts
```

```text
10.129.1.27 ignition.htb
```

With `http://ignition.htb/` resolving, the storefront loads normally. This is the correct way to handle a
redirect like this: if the name is not resolvable, every request to the IP will bounce and directory
enumeration will find nothing useful.

### Content discovery

The site is small, so directory and file brute forcing is the next step. Gobuster was run against the virtual
host with a common wordlist and a handful of extensions; the output is in the screenshots rather than the
transcript, but it surfaced an administrative login page.

<!-- screenshot: gobuster output against ignition.htb -->

### The admin panel

The administrative login presented a standard username and password form. Default credentials were tried
first, before any brute forcing:

```text
admin : qwerty123
```

The combination was accepted.

<!-- screenshot: admin login accepted with admin / qwerty123 -->

## Capturing the flag

The dashboard loaded straight onto the flag. No shell, no privilege escalation and no exploit were involved:
the box ends with a single working credential on an exposed administrative interface.

<!-- screenshot: flag shown on the admin dashboard -->

## Key takeaways

- **Follow redirects during enumeration.** A hostname-only deployment hides the entire application from anyone
  scanning by IP; adding the vhost to `/etc/hosts` is part of enumeration, not an afterthought.
- **Change default credentials at install time.** `admin` / `qwerty123` is a vendor default that appears on
  public lists, so it is equivalent to having no password at all.
- **Enforce a password policy and rate limiting on login forms.** A shared default with no lockout gives an
  attacker unlimited attempts; a policy that rejects common passwords removes this path entirely.
- **Put administrative panels behind a second control.** Restricting the login page by source address, VPN or
  single sign-on keeps it out of reach even when a weak password exists.
- **Brute forcing is a last resort.** Trying the documented default costs one request; a wordlist against a
  login form costs hours and usually triggers alarms.
