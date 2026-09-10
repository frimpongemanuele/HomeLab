# Lab Environment

## Overview

The HomeLab includes a dedicated environment for development, experimentation, and security testing.

Rather than performing experimental work directly on production-like infrastructure, a separate Proxmox LXC container is used as a disposable and recoverable testing environment.

The current lab workload is:

| Property | Configuration |
|---|---|
| Platform | Proxmox VE |
| Container ID | `106` |
| Name | WebApp |
| Type | Unprivileged LXC |
| CPU | 2 vCPU |
| Memory | 4 GB |
| Network | Home LAN |
| Purpose | Development / testing / security experimentation |

The environment provides a controlled location for experimenting with:

- Web applications
- Linux services
- Docker applications where appropriate
- Reverse proxies
- Databases
- APIs
- Application deployment
- Networking
- Authentication
- Security testing
- Monitoring
- Troubleshooting

The goal is to separate experimentation from the stable services that support the rest of the HomeLab.

---

# Architecture

The lab currently runs as an independent LXC container on the Proxmox host.

```text
                     Proxmox VE
                         │
          ┌──────────────┼──────────────┐
          │              │              │
      Stable         Services          Lab
     Workloads                          │
          │                             ▼
     HA / Jellyfin                  LXC 106
     Pi-hole etc.                    WebApp
                                        │
                               Development / Testing
                                        │
                           ┌────────────┼────────────┐
                           ▼            ▼            ▼
                       Web Apps       APIs       Security
                                                 Testing
```

The LXC boundary separates the lab filesystem and processes from the other HomeLab workloads.

However, the current network remains primarily flat.

This distinction is important:

> **Virtualization isolation does not automatically provide network isolation.**

---

# Why a Dedicated Lab?

Infrastructure experimentation often requires:

- Installing unfamiliar packages
- Modifying configuration files
- Running development servers
- Testing new software
- Opening temporary ports
- Running intentionally vulnerable applications
- Testing authentication
- Generating unusual network traffic
- Breaking and rebuilding services

Performing these activities directly on infrastructure such as Proxmox, Home Assistant, Pi-hole, or the Docker host would create unnecessary risk.

The lab therefore follows the principle:

```text
Experiment
    │
    ▼
Dedicated Lab
    │
    ├── Success ──► Document / Deploy Properly
    │
    └── Failure ──► Roll Back / Rebuild
```

The environment is designed to make failure acceptable.

---

# Why an LXC Container?

The current lab uses an LXC rather than a full VM.

This provides several advantages.

## Low Resource Overhead

The lab shares the Proxmox host kernel and therefore requires fewer resources than a complete virtual machine.

## Fast Deployment

LXC containers can be created, cloned, snapshotted, and restored quickly.

## Independent Filesystem

Changes inside the lab do not directly modify the filesystems of other workloads.

## Independent Resource Allocation

CPU and memory can be allocated specifically to the testing environment.

## Easy Recovery

The container can be snapshotted before significant experiments and restored if necessary.

---

# Unprivileged Container

LXC 106 is configured as an **unprivileged container**.

This means container user IDs are mapped to non-root user IDs on the Proxmox host.

Conceptually:

```text
Container
root (UID 0)
     │
     │ UID Mapping
     ▼
Proxmox Host
Unprivileged UID
```

This reduces the impact of certain container escape or filesystem-access scenarios compared with a privileged LXC.

However:

> An unprivileged LXC is still not equivalent to a full virtual machine security boundary.

LXC containers share the host kernel.

Higher-risk experiments may therefore require a dedicated VM instead.

---

# Lab Workload Categories

The environment can support several categories of testing.

## Web Application Development

Examples include:

```text
Frontend applications
Backend services
REST APIs
Authentication systems
Database-backed applications
Self-hosted web tools
```

A typical development architecture could be:

```text
Browser
   │
   ▼
Web Application
   │
   ├── API
   │
   └── Database
```

The environment can be used to understand the complete application lifecycle from deployment to monitoring.

---

# Deployment Testing

Before adding a new application to the main HomeLab, the lab can be used to evaluate:

- Installation procedure
- Dependencies
- Resource consumption
- Required ports
- Persistent storage
- Authentication
- Upgrade behavior
- Backup requirements
- Reverse-proxy compatibility
- Monitoring requirements

The workflow becomes:

```text
Discover Application
        │
        ▼
Deploy in Lab
        │
        ▼
Evaluate
        │
   ┌────┼────┐
   │    │    │
Security Resources Stability
   │    │    │
   └────┼────┘
        │
        ▼
Production Decision
```

This reduces the number of unknowns before introducing software into the main environment.

---

# Security Testing

The lab also provides a platform for defensive application-security experimentation.

Potential areas include:

- Authentication testing
- Authorization testing
- Input validation
- HTTP security headers
- TLS configuration
- Dependency vulnerabilities
- Logging
- Rate limiting
- Reverse-proxy configuration
- Network exposure
- OWASP Top 10 concepts

Examples of vulnerabilities that can be studied in controlled applications include:

```text
SQL Injection
Cross-Site Scripting
Broken Access Control
Command Injection
File Upload Vulnerabilities
Server-Side Request Forgery
Authentication Weaknesses
Security Misconfiguration
```

Testing is restricted to systems owned by or explicitly authorized for the HomeLab.

---

# Security Testing Workflow

A structured testing process can follow:

```text
Deploy Test Application
          │
          ▼
Identify Attack Surface
          │
          ▼
Test Application
          │
          ▼
Observe Logs / Traffic
          │
          ▼
Implement Mitigation
          │
          ▼
Retest
          │
          ▼
Document Result
```

This transforms the lab from a simple application host into a learning environment for both offensive testing concepts and defensive remediation.

---

# Snapshots and Rollback

One of the most useful features of the lab is the ability to create recovery points before experiments.

Example workflow:

```text
Stable Lab
    │
    ▼
Create Snapshot
    │
    ▼
Run Experiment
    │
 ┌──┴───┐
 │      │
Good   Broken
 │      │
 ▼      ▼
Keep   Roll Back
```

A Proxmox snapshot can be created before a high-risk configuration change.

Example:

```bash
pct snapshot 106 before-test
```

If the experiment causes problems, the container can be rolled back rather than manually repaired.

Snapshots are temporary recovery mechanisms and do not replace backups.

---

# Disposable Infrastructure

An important principle of the lab is that the environment should remain replaceable.

Where possible:

```text
Configuration
     +
Documentation
     +
Source Code
     +
Deployment Instructions
     =
Reproducible Environment
```

The objective is to avoid treating the lab container itself as irreplaceable infrastructure.

If the environment becomes excessively modified or unstable, rebuilding it can be preferable to attempting to repair every historical change.

---

# Current Network Architecture

At present, the lab is connected to the same primary LAN as much of the rest of the HomeLab.

```text
               192.168.1.0/24
                     │
        ┌────────────┼────────────┐
        │            │            │
   Management     Services       Lab
        │            │            │
     Proxmox       Jellyfin     WebApp
                   Docker       LXC 106
```

This is convenient for development but creates an important security limitation.

A compromised lab application could potentially attempt to communicate with other systems on the LAN.

Therefore, the current LXC separation should not be interpreted as complete security isolation.

---

# Current Security Boundary

The current lab benefits from:

- Dedicated LXC
- Unprivileged container configuration
- Separate filesystem
- Separate process namespace
- Dedicated resource allocation
- Independent application environment
- Snapshot capability

However, it does **not yet** have:

- Dedicated VLAN
- Inter-VLAN firewall policy
- Dedicated lab firewall zone
- Full kernel isolation
- Complete traffic inspection
- IDS/IPS monitoring

This is intentionally documented as a current architectural limitation.

---

# Target Lab Architecture

The planned design places experimental workloads in a dedicated network security zone.

```text
                  Router / Firewall
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
        Management    Services     Lab VLAN
           VLAN         VLAN          │
              │          │            ▼
          Proxmox     Jellyfin      WebApp
          Admin       Docker       Testing
                                      │
                                      ▼
                                  Internet
```

The Lab VLAN should have restricted access to trusted infrastructure.

---

# Target Firewall Policy

A future firewall policy could follow:

| Source | Destination | Policy |
|---|---|---|
| Management | Lab | Allow administration |
| Lab | Management | Deny |
| Lab | Services | Deny by default |
| Lab | IoT | Deny |
| Lab | Guest | Deny |
| Lab | Internet | Allow as required |
| VPN Admin | Lab | Allow |
| Internet | Lab | Deny unsolicited inbound |

The general rule would be:

```text
Trusted Admin
     │
     ▼
    Lab

Lab
 │
 ├──► Internet     ALLOW where required
 │
 ├──► Management   DENY
 │
 ├──► Services     DENY by default
 │
 └──► IoT          DENY
```

This would substantially reduce the blast radius of a compromised experimental application.

---

# Lab vs Production-Like Services

The HomeLab distinguishes between stable and experimental workloads.

| Stable Infrastructure | Lab |
|---|---|
| Proxmox | Experimental applications |
| Home Assistant | Development servers |
| Jellyfin | Vulnerable test applications |
| Pi-hole | Temporary databases |
| Tailscale | API experiments |
| Homepage | Security testing |
| Monitoring | Deployment experiments |
| Docker Media Stack | Disposable services |

The goal is not to claim enterprise production availability.

Instead, the distinction identifies which systems should remain stable and which systems are intentionally allowed to change frequently.

---

# Reverse Proxy Testing

The lab can also be used to test reverse-proxy configurations before applying similar concepts to stable services.

A typical test flow is:

```text
Client
   │
   ▼
Reverse Proxy
   │
   ▼
Lab Application
```

Areas that can be evaluated include:

- Hostname routing
- TLS
- HTTP headers
- Authentication
- Access controls
- WebSocket support
- Proxy timeouts
- Application compatibility

Once validated, the design can be documented before being introduced elsewhere.

---

# Monitoring the Lab

Experimental environments should still be observable.

The HomeLab monitoring stack can provide visibility into:

- CPU usage
- Memory consumption
- Disk usage
- Network activity
- Service availability
- Unexpected restarts

Conceptually:

```text
Lab LXC
   │
   ├── Metrics ──► Monitoring Stack
   │
   ├── Health ───► Uptime Monitoring
   │
   └── Logs ─────► Future Central Logging
```

Monitoring is particularly useful during testing because it shows how applications behave under unusual conditions.

---

# Logging

Application logs are important for both development and security testing.

Useful sources include:

```text
Web server logs
Application logs
Authentication logs
Linux system logs
Reverse proxy logs
Database logs
```

A future centralized logging platform could collect these events for analysis.

Potential platforms include:

- Grafana Loki
- Graylog
- Elastic Stack

This would allow lab experiments to be correlated with infrastructure events.

---

# Security Monitoring Roadmap

The lab is a suitable environment for testing security-monitoring technologies before wider deployment.

Potential tools include:

## Wazuh

Possible uses:

- Host intrusion detection
- File integrity monitoring
- Security event collection
- Vulnerability information
- Alerting

## Suricata

Possible uses:

- Network intrusion detection
- Signature-based detection
- Network telemetry
- Traffic analysis

The lab could generate controlled security events that allow detection rules and dashboards to be evaluated safely.

---

# Example Future Security Lab

A more advanced environment could evolve into:

```text
                   Security Lab
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
      Attacker        Target       Monitoring
         VM             VM             │
          │              │             ├── Wazuh
          └──────► Test Network ◄──────┤
                                       └── Suricata
```

This would provide stronger isolation and more realistic security-testing capabilities than the current single LXC environment.

A dedicated virtual network could prevent experimental traffic from reaching the main HomeLab.

---

# When to Use a VM Instead

LXC is appropriate for many development workloads, but it is not ideal for every experiment.

A full VM should be preferred when testing:

- Unknown or untrusted operating-system images
- Kernel-level behavior
- Malware analysis
- High-risk exploitation
- Network appliances
- Alternative operating systems
- Experiments requiring stronger isolation
- Software requiring its own kernel

Conceptually:

```text
Low / Moderate Risk
        │
        ▼
Unprivileged LXC

Higher Risk
        │
        ▼
Dedicated VM

Very High Risk
        │
        ▼
Dedicated Isolated Lab
```

The security boundary should match the risk of the experiment.

---

# Secrets and Test Data

The lab should not contain unnecessary production credentials.

Examples of information that should not be copied into experimental applications include:

```text
Proxmox administrator credentials
Real API tokens
Tailscale authentication keys
Home Assistant tokens
Samba passwords
Private SSH keys
Cloud credentials
```

Test applications should use:

```text
Dummy accounts
Temporary credentials
Test API keys
Synthetic data
```

where practical.

This reduces the impact of accidental credential disclosure during development or testing.

---

# Troubleshooting Methodology

The lab is also useful for developing a systematic troubleshooting process.

A typical workflow is:

```text
Application Failure
        │
        ▼
Process Running?
        │
        ▼
Port Listening?
        │
        ▼
Network Reachable?
        │
        ▼
Dependencies Available?
        │
        ▼
Configuration Valid?
        │
        ▼
Logs
        │
        ▼
Root Cause
```

Useful Linux commands include:

```bash
systemctl status <service>
```

```bash
journalctl -u <service>
```

```bash
ss -lntp
```

```bash
ip addr
```

```bash
ip route
```

```bash
df -h
```

```bash
free -h
```

```bash
top
```

or:

```bash
btop
```

The objective is to troubleshoot methodically rather than repeatedly changing configuration without identifying the underlying failure.

---

# Documentation Workflow

Successful experiments should produce documentation.

The preferred lifecycle is:

```text
Experiment
    │
    ▼
Working Configuration
    │
    ▼
Sanitize Secrets
    │
    ▼
Document Architecture
    │
    ▼
Commit to Git
```

This turns temporary experiments into reusable knowledge.

It also makes the HomeLab repository a record of both infrastructure design and technical learning.

---

# Design Principles

## Keep Experiments Away from Core Infrastructure

Testing should not require modifying stable workloads unnecessarily.

## Make Failure Recoverable

Snapshots and rebuild procedures make experimentation safer.

## Use the Minimum Required Privileges

Experiments should not automatically receive access to sensitive infrastructure.

## Match Isolation to Risk

LXC is appropriate for many workloads, while higher-risk testing should use stronger isolation.

## Document Current Limitations

The current flat network must not be presented as equivalent to a segmented security lab.

## Promote Successful Experiments

Once a design is stable and understood, it can be introduced into the main HomeLab using documented procedures.

---

# Future Improvements

Planned improvements include:

- Dedicated Lab VLAN
- Inter-VLAN firewall rules
- Dedicated virtual test networks
- Additional disposable VMs
- Separate attacker and target systems
- Wazuh integration
- Suricata monitoring
- Centralized logging
- Automated lab provisioning
- Infrastructure-as-Code
- Ansible
- Vulnerability scanning
- Application security testing
- OWASP testing environments
- Container security testing
- CI/CD experimentation
- Automated deployment pipelines
- Snapshot automation
- Reusable VM/LXC templates

---

# Skills Demonstrated

The lab environment provides practical experience with:

- Proxmox VE
- Linux containers
- Unprivileged LXC
- Linux administration
- Resource allocation
- Networking
- Web application deployment
- API deployment
- Reverse proxies
- Troubleshooting
- Snapshot and rollback workflows
- Security testing concepts
- Threat modeling
- Network segmentation design
- Firewall policy design
- Logging
- Monitoring
- Application security
- Infrastructure documentation

---

# Key Takeaways

The lab environment provides a controlled location for learning through experimentation without deliberately destabilizing the core HomeLab services.

The current design combines:

```text
Proxmox
   +
Unprivileged LXC
   +
Snapshots
   +
Development
   +
Security Testing
   +
Monitoring
```

while acknowledging that the current flat LAN limits the strength of the security boundary.

The long-term goal is to evolve the environment into a segmented and observable security lab where experimental systems can be deployed, attacked, monitored, destroyed, and rebuilt without affecting trusted infrastructure.

---

# Related Documentation

- [Architecture](architecture.md)
- [Networking](networking.md)
- [Cybersecurity](cybersecurity.md)
- [Monitoring](monitoring.md)
- [Backup & Recovery](backup-recovery.md)
