Yooo guys, it's ju4dy the landlord again.

# Explosion — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 0 |
| **Difficulty** | Very easy |
| **Operating system** | Windows |
| **Target IP** | `10.129.1.13` |
| **Skills** | Service enumeration, Remote Desktop Protocol, unauthenticated RDP access |

## Overview

Explosion is a very easy Windows machine that introduces the Remote Desktop Protocol (RDP). The box is a
straightforward demonstration of what happens when an RDP service is left reachable without requiring a
password: no exploit and no credential attack are needed, only a client such as `xfreerdp` and the address of
the host.

## Enumeration

A default TCP scan with service and version detection is enough to characterize the host.

```bash
nmap -sVC 10.129.1.13
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 09:17 +0100
Nmap scan report for 10.129.1.13
Host is up (0.43s latency).
Not shown: 995 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: EXPLOSION
|   NetBIOS_Domain_Name: EXPLOSION
|   NetBIOS_Computer_Name: EXPLOSION
|   DNS_Domain_Name: Explosion
|   DNS_Computer_Name: Explosion
|   Product_Version: 10.0.17763
|_  System_Time: 2026-10-14T08:17:56+00:00
| ssl-cert: Subject: commonName=Explosion
| Not valid before: 2026-10-13T08:03:00
|_Not valid after:  2027-04-14T08:03:00
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

The scan reveals a standard Windows footprint. Port `135` and `139`/`445` belong to RPC and SMB, `5985` is
WinRM (`Microsoft-HTTPAPI/2.0`), and `3389` is the Remote Desktop service. The `rdp-ntlm-info` script also
leaks the computer name (`EXPLOSION`) and the build number `10.0.17763`, which corresponds to Windows Server
2019 / Windows 10 1809.

One detail worth noting is the SMB configuration: message signing is *enabled but not required*. That does not
matter for this box, but it is the kind of weakness that becomes relevant during lateral movement.

## Foothold — connecting over RDP

Because RDP is exposed, the first thing to try is a plain client connection. `xfreerdp3` is invoked with only
a target host and a username; it does not prompt for a password:

```bash
xfreerdp3 /v:10.129.1.13 /u:Administrator
```

On a properly configured host this would fail with an authentication error. Here the session opens directly
onto the Administrator desktop, which means the account has no password and the RDP service accepts it. In
other words, the misconfiguration — not a vulnerability in RDP — is the way in.

A few useful client flags if the connection misbehaves:

```bash
xfreerdp3 /v:10.129.1.13 /u:Administrator /cert:ignore /dynamic-resolution
```

- `/cert:ignore` accepts the self-signed certificate shown in the scan output.
- `/dynamic-resolution` lets the desktop resize with the window.
- `rdesktop` and the Remmina GUI are equivalent alternatives.

## Capturing the flag

Inside the desktop session the flag is already present on the Administrator desktop. Reading it and submitting
it on the Starting Point page completes the box.

<!-- screenshot: flag.txt on the Administrator desktop -->

## Key takeaways

- **Passwordless privileged accounts are a critical finding.** The Administrator account having no password
  turns an administrative protocol into an open door.
- **RDP should require Network Level Authentication** and should never be reachable from untrusted networks
  without a jump host or VPN. Restrict `3389/tcp` at the firewall as a defense-in-depth measure.
- **Banner data is free intelligence.** `rdp-ntlm-info` handed over the hostname and OS build without any
  credentials, which is useful for choosing follow-up tooling.
- **WinRM on `5985` is the natural next step** once a password is known — `evil-winrm` provides a shell without
  the overhead of a full desktop session.
- **SMB signing is enabled but not required**, which is worth flagging in a report: it permits SMB relay
  attacks against the host if an attacker can coerce authentication.
