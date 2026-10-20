# Hack The Box — Starting Point writeups

Notes from working through the Hack The Box Starting Point track, rewritten as walkthroughs. Every entry
follows the same structure: enumeration, entry point, escalation, and the defensive lesson behind it.

The commands are the ones that actually ran during the session, including the missteps that are worth
remembering.

## Tier 0 — single services

The first tier is about recognizing one service and knowing its default configuration.

- **[Explosion](tier-0/explosion.md)** — Windows, RDP answering without a password, accessed with `xfreerdp`.
- **[Preignition](tier-0/preignition.md)** — a stock nginx page hiding an admin login that accepts `admin:admin`.
- **[Mongod](tier-0/mongod.md)** — MongoDB on `27017` with authentication never enabled, read through `mongosh`.
- **[Synced](tier-0/synced.md)** — an `rsync` daemon publishing an anonymous module containing the flag.

## Tier 1 — web, SMB and credentials

The second tier chains a finding into access: a leaked secret, a trusted client value, a service that
authenticates to us.

- **[Responder](tier-1/responder.md)** — remote file inclusion to an SMB callback, NTLMv2 capture, cracking, WinRM.
- **[Three](tier-1/three.md)** — virtual host discovery leading to an S3 bucket that is also the web root.
- **[Funnel](tier-1/funnel.md)** — anonymous FTP mail backup, a password that was never changed, SSH spray, port forward.
- **[Bike](tier-1/bike.md)** — Handlebars template injection in a contact form, escalated to Node.js command execution.
- **[Ignition](tier-1/ignition.md)** — a hostname-only deployment and a default administrative password.
- **[Pennyworth](tier-1/pennyworth.md)** — Jenkins left on default credentials, exploited through the Script Console.
- **[Tactics](tier-1/tactics.md)** — SMB enumeration and Impacket `psexec` against a weakly configured local administrator.

## Tier 2 — multi-step chains

The third tier combines several small weaknesses, where no single bug is enough on its own.

- **[Vaccine](tier-2/vaccine.md)** — anonymous FTP backup, two cracked hashes, SQL injection to an OS shell, reused credentials.
- **[Oopsie](tier-2/oopsie.md)** — IDOR and cookie tampering to reach an upload form, then a SUID `PATH` hijack.

## How to read these

Every entry starts with the scan, follows the path that actually worked, and ends with the flag and the
defensive lesson. Commands are left as they were run, including the options that mattered and the ones that
did not.
