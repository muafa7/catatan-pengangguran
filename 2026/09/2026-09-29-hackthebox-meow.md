# Hack The Box — Meow

Today I finished **Hack The Box — Meow**, my first Starting Point machine.

This box was very simple compared to the web pentesting lab I did before, but I think that's the point.

There wasn't really an exploit to build or a complicated vulnerability chain.

The challenge was more about learning the basic pentesting workflow:

```text
connect
   ↓
enumerate
   ↓
identify exposed service
   ↓
understand the service
   ↓
test authentication
   ↓
gain access
```

## 1. Start with connectivity

Before doing anything to a target, I need to make sure I can actually reach it.

```bash
ping <TARGET_IP>
```

Simple, but useful.

If the machine isn't reachable, there's no point debugging Nmap, Telnet, or anything further up the chain yet.

---

## 2. Enumeration with Nmap

Next was finding out what the machine was exposing.

```bash
nmap -sV <TARGET_IP>
```

The important result was:

```text
23/tcp open  telnet
```

This was a good reminder that I shouldn't jump straight into trying random exploits.

First answer:

> What is actually running on this machine?

In this case there was basically one obvious direction to investigate:

```text
Target
  ↓
TCP 23
  ↓
Telnet
```

---

## 3. Telnet

I had heard of Telnet before, but this challenge made its role much clearer.

I connected using:

```bash
telnet <TARGET_IP>
```

Telnet gives remote terminal access to another machine.

The problem is that Telnet is an old protocol and does not provide the encrypted communication I would expect from something like SSH.

So seeing Telnet exposed is already something worth investigating.

But an insecure protocol alone wasn't the main problem with this machine.

The bigger issue was the authentication configuration.

---

## 4. Always test the simple things

The Telnet server asked for a username.

Instead of immediately searching for an exploit or CVE, I tried:

```text
root
```

Then when it asked for a password, the account accepted a **blank password**.

That immediately gave me access to the machine as:

```text
root
```

This is probably the main thing I want to remember from Meow:

> Don't skip simple misconfigurations because I'm looking for something more complicated.

Sometimes the vulnerability isn't:

```text
memory corruption
RCE exploit
CVE
buffer overflow
```

Sometimes it's literally:

```text
root
+
no password
```

---

## 5. There was no privilege escalation

Normally I think of a pentest flow like:

```text
enumeration
   ↓
initial foothold
   ↓
privilege escalation
   ↓
root
```

But Meow was different.

The initial foothold was already:

```text
root
```

So:

```text
Telnet
   ↓
root + blank password
   ↓
root shell
```

There was nothing left to escalate.

This helped me understand that **privilege escalation isn't a mandatory step**.

It's something I need when the access I initially obtain has lower privileges than what I'm trying to reach.

---

## What I learned

### Enumeration comes before exploitation

Running Nmap wasn't just something to do because a tutorial told me to.

The result determined the entire direction of the test.

```text
scan
  ↓
discover Telnet
  ↓
investigate Telnet
```

Without enumeration, I'm basically guessing.

---

### Check for weak authentication before overcomplicating things

Before looking for sophisticated exploits, check obvious security mistakes:

- blank passwords
- default credentials
- weak credentials
- anonymous access
- unnecessary exposed services

A complicated exploit isn't necessary if the front door is already open.

---

### A service being reachable matters

It wasn't enough that Telnet existed on the machine.

The important combination was:

```text
Telnet exposed
        +
remote root login allowed
        +
blank root password
```

Together, those configuration mistakes resulted in complete system access.

---

## One thing that clicked

Meow made the basic pentesting loop clearer to me:

```text
What can I reach?
        ↓
What ports are open?
        ↓
What services are running?
        ↓
How does that service normally work?
        ↓
How is this instance configured?
        ↓
Is there something insecure about it?
```

That's a better way to think than immediately asking:

> "What exploit should I run?"

The tool isn't the methodology.

`nmap` just helped me answer a question.

`telnet` just helped me interact with the service.

The actual process was understanding what was exposed and noticing that its configuration was insecure.

---

## Commands worth remembering

Check connectivity:

```bash
ping <TARGET_IP>
```

Enumerate services:

```bash
nmap -sV <TARGET_IP>
```

Connect to Telnet:

```bash
telnet <TARGET_IP>
```

Check the current user after getting a shell:

```bash
whoami
```

For Meow, the interesting result was:

```text
root
```

---

## Final takeaway

Meow was easy technically, but useful conceptually.

My takeaway isn't:

> "Telnet + root = flag."

It's:

> **Enumerate first, understand what is exposed, and test the simplest security failures before searching for complicated exploits.**

A system doesn't need an advanced vulnerability to be completely compromised.

A bad configuration can be enough.