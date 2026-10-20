Yooo guys, it's ju4dy the landlord again.

# Tactics — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 1 |
| **Difficulty** | Very easy |
| **Operating system** | Windows |
| **Target IP** | `10.129.57.201` |
| **Skills** | Full-port scanning, SMB enumeration, null sessions, Impacket psexec |

## Overview

Tactics is a very easy Windows machine whose only exposed services are SMB and RPC. There is no web
application and no obvious entry point, which makes it a pure SMB exercise: enumerate the shares, discover
that the service accepts a session without a usable password, and then turn that access into a shell with
Impacket's `psexec`.

The box is useful practice for two reasons. First, it shows that an all-ports scan is worth the wait even when
almost everything is filtered. Second, it demonstrates that SMB reachability plus weak account configuration
is enough for full administrative compromise — no exploit code required.

## Enumeration

```bash
nmap -sV -sC -O -p- -T4 10.129.57.201
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-15 07:32 +0100
Stats: 0:05:32 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 86.97% done; ETC: 07:38 (0:00:50 remaining)
Nmap scan report for 10.129.57.201
Host is up (0.51s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE SERVICE       VERSION
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (91%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Windows Server 2019 (91%), Microsoft Windows 10 1903 - 21H1 (85%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-10-15T06:39:13
|_  start_date: N/A

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 454.91 seconds
```

Three observations shape everything that follows:

- **Only one protocol family is open.** `135` is the RPC endpoint mapper, `139` is NetBIOS session service and
  `445` is SMB over TCP. There is no HTTP, no RDP and no WinRM, so any entry point has to come through SMB.
- **The rest of the surface is filtered, not closed.** `65532` ports responded with no reply at all, which is
  why the scan took over seven minutes. A fast default scan would have shown the same three ports, but the
  all-ports run confirms there is nothing hiding on a high port.
- **SMB signing is enabled but not required.** Signing protects the integrity of SMB traffic and, more
  importantly here, it is what makes SMB relay attacks possible when it is optional. On this box it is a
  contributing weakness rather than the path in.

## SMB enumeration

The next step is a null session — an SMB connection with no username and no password. SMB servers that permit
it will happily list their shares, which is often enough to learn the machine's purpose:

```bash
smbclient -L //10.129.57.201/ -N
```

`-N` tells `smbclient` to send no password, so an accepted connection here means the server allows
unauthenticated enumeration. The same listing can be produced by Impacket when `smbclient` is unavailable:

```bash
impacket-smbclient -no-pass guest@10.129.57.201
```

Once a share name is known, it can be mounted for browsing:

```bash
smbclient //10.129.57.201/<share> -N
```

Inside the session the familiar `ls`, `cd` and `get` commands work as they do on a remote shell. Any files
found this way are worth downloading and reading locally, because configuration files, scripts and backups
frequently contain credentials.

## Foothold

### 1. `smbclient`

The first method in the notes is interactive: connect with `smbclient`, enumerate the directory tree of the
reachable share and pull down anything readable. `smbclient` is ideal for this because it requires no
preparation and proves the access level before anything more invasive is attempted.

A recursive download of the whole share keeps the results in one place for offline review:

```bash
smbclient //10.129.57.201/<share> -N -c 'recurse ON; prompt OFF; mget *'
```

If the session is read-only, that is still valuable: the contents usually reveal accounts, hostnames or
software versions. If it is writable, the usual next thought is dropping a payload — but a cleaner route to
code execution is the one below.

### 2. Impacket `psexec`

The second method abandons interactive browsing and goes straight for a shell. Impacket's `psexec` implements
the PsExec technique: it authenticates over SMB, copies a small service binary to `ADMIN$`, registers it as a
Windows service and starts it. Because Windows services run as `SYSTEM` by default, the resulting shell is
`NT AUTHORITY\SYSTEM`.

```bash
impacket-psexec Administrator@10.129.57.201
```

When the password prompt appears, an empty password is submitted by pressing Enter. That is the weakness the
box is built around: the local Administrator account has no password, so an SMB client authenticating as
`Administrator` is accepted without a credential. The shell that comes back is running as
`NT AUTHORITY\SYSTEM`, because the service the tool creates is started by the Windows service manager under
the local system account.

Two details make this possible beyond the blank password. The account must be a local administrator, since
writing to `ADMIN$` and creating a service both require administrative rights, and SMB signing being optional
means the authentication exchange can be abused or relayed. Hash-based authentication works the same way when
a hash is available instead of a password:

```bash
impacket-psexec -hashes :<NThash> Administrator@10.129.57.201
```

`impacket-smbexec` and `impacket-wmiexec` are the quieter alternatives: `smbexec` avoids dropping a binary by
creating a service that runs individual commands, and `wmiexec` uses WMI instead of the service manager, which
leaves fewer artifacts but produces a less interactive shell.

## Finishing the box

The `psexec` session came back as `NT AUTHORITY\SYSTEM` and the flag was read from the Administrator desktop.
There is nothing further to escalate on Tactics — SYSTEM is the top of the local stack, which is what makes a
blank password on a local administrator so damaging.

<!-- screenshot: smbclient share listing and interactive session -->
<!-- screenshot: impacket-psexec returning an NT AUTHORITY\SYSTEM shell -->

## Key takeaways

- **A blank password on a local administrator is a full compromise.** It defeats the purpose of authentication
  entirely; the account should be disabled or given a long, unique password, and local Administrator accounts
  should be managed with LAPS.
- **SMB signing should be required, not optional.** With signing enforced, an attacker who captures or relays
  authentication cannot reuse it, and the relay half of this class of attack disappears.
- **Null sessions should be disabled.** Restricting anonymous share and pipe enumeration removes the
  reconnaissance step that tells an attacker which share to open.
- **Port filtering hides nothing from an all-ports scan.** The scan took minutes because the host dropped
  packets, but the three open ports were still found. From the defender's side, filtering must be paired with
  host hardening rather than treated as isolation.
- **`psexec` leaves loud artifacts.** A new service, a dropped binary in `ADMIN$` and event log entries
  (`7045`, `4624` type 3) are all detectable; monitoring those sources catches this technique quickly.
