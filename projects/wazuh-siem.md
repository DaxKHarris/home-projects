# Wazuh SIEM

**Status: In Progress**

## Project Goals

- **Visibility** — centralized security monitoring across all owned devices
- **Early detection** — identify threats on the home network before they become real incidents
- **Hands-on experience** — build practical SIEM skills in a real environment, not just a lab

## Background

Studied Wazuh on TryHackMe before deploying it on the home network. The goal was to move beyond theoretical knowledge and operate a real SIEM against real infrastructure.

## Why Wazuh

Wazuh is open source, actively maintained, and used in real enterprise environments. It covers log aggregation, intrusion detection, vulnerability detection, and alerting — the core functions of a SIEM. It was the natural choice for a home setup that needed to mirror production-grade security monitoring.

## Deployment

Wazuh runs as three separate containers following the official compose configuration:

- **wazuh-manager** — receives and processes events from agents
- **wazuh-indexer** — stores and indexes security events (OpenSearch-based)
- **wazuh-dashboard** — visualization and alerting interface

**Podman** was chosen over Docker deliberately. Podman runs containers rootless by default — containers operate without root privileges on the host. With Docker, a container escape gives an attacker root on the host immediately. With rootless Podman, a container escape lands in an unprivileged user context. For a security-focused setup, running a container runtime that requires explicitly opting into security made no sense.

The stack runs on the Jellyfin server, which runs Debian minimal.

## Monitoring Scope

Planned agents across all owned devices:

- Jellyfin home media server
- Raspberry Pi (reverse proxy + VPN endpoint)
- Personal machines

## Known Issues & Design Conflicts

### SIEM on a Sleeping Host

The Jellyfin server sleeps when not in use as part of its power management design. A SIEM must be always-on — any downtime is a blind spot. This is a direct architectural conflict.

The current host is not suitable for Wazuh long term. The correct solution is a dedicated always-on machine. This is a known issue pending a hardware decision.

## Pending

- Deploy Wazuh agents on all owned devices
- Resolve always-on host conflict — identify dedicated hardware
- Configure alerting rules relevant to home network threat model
