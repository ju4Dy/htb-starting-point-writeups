# Hack The Box — Starting Point writeups

Working notes from the Hack The Box **Starting Point** track, cleaned up into walkthroughs. Each entry follows
the same shape: what the scan showed, how the box was entered, what the escalation was, and what the box
teaches about defending against it.

These are written after the fact from my own session notes, so the commands are the ones that actually ran —
including the dead ends worth remembering.

## Index

| Machine | Tier | OS | Difficulty | Core technique |
| --- | --- | --- | --- | --- |
| [Explosion](writeups/tier-0/explosion.md) | 0 | Windows | Very easy | Unauthenticated RDP with `xfreerdp` |
| [Preignition](writeups/tier-0/preignition.md) | 0 | Linux | Very easy | Directory brute forcing, default `admin:admin` |
| [Mongod](writeups/tier-0/mongod.md) | 0 | Linux | Very easy | MongoDB with authentication disabled |
| [Synced](writeups/tier-0/synced.md) | 0 | Linux | Very easy | Anonymous `rsync` module |
| [Responder](writeups/tier-1/responder.md) | 1 | Windows | Very easy | RFI to UNC path, Responder, NTLMv2 cracking, WinRM |
| [Three](writeups/tier-1/three.md) | 1 | Linux | Very easy | Vhost discovery, writable S3 bucket as web root |
| [Funnel](writeups/tier-1/funnel.md) | 1 | Linux | Very easy | Anonymous FTP, leaked password, SSH spray, port forward |
| [Bike](writeups/tier-1/bike.md) | 1 | Linux | Very easy | Handlebars SSTI to Node.js RCE |
| [Ignition](writeups/tier-1/ignition.md) | 1 | Linux | Very easy | Vhost redirect, default admin credentials |
| [Pennyworth](writeups/tier-1/pennyworth.md) | 1 | Linux | Very easy | Jenkins Script Console, Groovy reverse shell |
| [Tactics](writeups/tier-1/tactics.md) | 1 | Windows | Very easy | SMB null session, Impacket `psexec` |
| [Vaccine](writeups/tier-2/vaccine.md) | 2 | Linux | Easy | Backup cracking, MD5 cracking, `sqlmap --os-shell` |
| [Oopsie](writeups/tier-2/oopsie.md) | 2 | Linux | Easy | IDOR, cookie tampering, upload, SUID PATH hijack |

Thirteen boxes, each written from a finished session: starting scan through to the flag. Every entry follows
the same shape so they can be read side by side.

## Repository layout

```text
.
├── README.md            # this page
├── mkdocs.yml           # optional static-site build
└── writeups
    ├── index.md         # landing page for the generated site
    ├── tier-0/          # very easy: single-service introductions
    ├── tier-1/          # very easy: web, SMB and credential work
    └── tier-2/          # easy: multi-step chains
```

The original Obsidian notes stay local and are excluded through `.gitignore` — they contain unredacted flags
and image links that only resolve inside the vault.

## A note on flags

Flag values are redacted as `[flag redacted]`. They add nothing to a walkthrough and there is no reason to
republish them. Credentials, hashes and payloads are kept as they were used, because those are the part worth
reading.

Screenshots from the original notes are not included. Each writeup marks the places they belong with an
invisible HTML comment, for example:

```html
<!-- screenshot: db.flag.find() output with the flag document -->
```

Dropping an image at `writeups/assets/<machine>/<name>.png` and replacing the comment with
`![description](assets/<machine>/<name>.png)` restores the visual evidence without touching the prose.

## Publishing

The content is plain Markdown and renders correctly on GitHub as it stands.

**GitHub Pages** — the simplest route, no extra service required. The repository is initialized, committed and
already points at `https://github.com/ju4Dy/htb-starting-point-writeups.git`, so publishing is: create the empty
repository on GitHub, push, then optionally build the site.

```bash
# create github.com/ju4Dy/htb-starting-point-writeups (public, without a README)
git push -u origin main

# optional: publish the Markdown as a site
pip install mkdocs-material
mkdocs gh-deploy          # builds and pushes to the gh-pages branch
```

Then set *Settings → Pages → Source* to the `gh-pages` branch. `mkdocs.yml` already lists every page in the
navigation order; switch `theme: mkdocs` to `theme: material` once the package is installed for a nicer layout.

**Alternatives** — Vercel or Cloudflare Pages both build a MkDocs site from the repository and give preview
deployments per commit, which is useful if the collection keeps growing. For cross-posting individual boxes,
dev.to and Medium accept Markdown directly, though code blocks and tables need a quick check after import.

## Ethics

Every machine here is a Hack The Box lab target on an isolated network. The techniques are documented for
defenders and for people learning the same material; nothing in this repository should be pointed at systems
you do not own or have explicit permission to test.
