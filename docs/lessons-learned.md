# Lessons Learned

## Overview

Building the HomeLab has involved much more than installing individual applications.

The project required decisions involving:

- Virtualization
- Linux administration
- Storage
- Networking
- Containers
- Smart-home infrastructure
- Media services
- Monitoring
- Remote access
- Cybersecurity
- Backup and recovery

Several of the most valuable lessons came from problems rather than successful installations.

Configuration errors, storage incidents, compatibility problems, and architectural limitations all influenced how the environment evolved.

This document summarizes the main engineering lessons learned while designing, operating, troubleshooting, and improving the HomeLab.

---

# 1. Architecture Matters More as the Lab Grows

The HomeLab started with a relatively small number of workloads.

As additional services were introduced, infrastructure organization became increasingly important.

The current environment separates major workloads using:

```text
Proxmox
   │
   ├── VM
   │    └── Home Assistant
   │
   ├── Dedicated LXCs
   │    ├── Jellyfin
   │    ├── Tailscale
   │    ├── Homepage
   │    ├── Pi-hole
   │    └── WebApp Lab
   │
   └── Docker LXC
        └── Containerized applications
```

The key lesson was:

> **Not every application should automatically become another Docker container.**

Some services benefit from dedicated workloads because they have different:

- Security requirements
- Hardware requirements
- Availability requirements
- Networking roles
- Update cycles
- Failure domains

For example, Home Assistant remains independent from Docker, while Pi-hole is separated because DNS is a critical network dependency.

---

# 2. Virtualization Isolation Is Not Network Isolation

VMs, LXC containers, and Docker containers provide useful workload separation.

However, they do not automatically create network security boundaries.

The current HomeLab primarily operates on:

```text
192.168.1.0/24
```

Therefore:

```text
Separate VM
     ≠
Separate Network

Separate LXC
     ≠
Separate Trust Zone
```

A compromised service may still attempt to communicate with other systems on the same LAN.

This became particularly important after introducing the WebApp testing environment and additional IoT devices.

The resulting architectural direction is to introduce dedicated zones for:

```text
Management
Services
IoT
Lab
Guest
```

with firewall rules controlling communication between them.

### Lesson

> **Compute isolation and network segmentation solve different problems.**

Both are required for stronger defense-in-depth.

---

# 3. LXC and VM Isolation Are Not Equivalent

LXC containers are lightweight and efficient, making them excellent for many HomeLab services.

However, containers share the Proxmox host kernel.

A virtual machine provides a separate guest kernel and therefore a stronger isolation boundary.

This influenced the workload strategy:

```text
Normal Service
     │
     ▼
LXC

Sensitive / Appliance Workload
     │
     ▼
VM

Higher-Risk Experiment
     │
     ▼
Dedicated VM / Isolated Lab
```

Home Assistant, for example, runs as a full VM, while many infrastructure services run efficiently as LXCs.

### Lesson

> **The isolation technology should match the risk and requirements of the workload.**

---

# 4. Docker Inside LXC Introduces Security Trade-Offs

Running Docker inside an LXC provides convenient separation between the Docker environment and the Proxmox host.

However, nested containerization introduced compatibility issues.

The Docker LXC required configuration such as:

```text
features: nesting=1,keyctl=1
lxc.apparmor.profile: unconfined
```

Some Docker containers may additionally require:

```yaml
security_opt:
  - apparmor=unconfined
```

Relaxing AppArmor confinement solved compatibility problems, but weakened one layer of security.

The important lesson was not simply how to make Docker work.

It was understanding the trade-off:

```text
Compatibility
     ↑
     │
     │
     ▼
Confinement
```

### Lesson

> **A configuration that fixes a technical problem can simultaneously weaken a security control.**

Those trade-offs should be understood and documented.

---

# 5. DNS Is Infrastructure, Not Just Another Application

Pi-hole initially appears similar to any other self-hosted service.

In practice, DNS is a foundational dependency.

If the DNS service fails:

```text
Client
   │
   ▼
DNS unavailable
   │
   ▼
Hostnames fail
   │
   ▼
Many services appear unavailable
```

For this reason, Pi-hole was placed in its own dedicated LXC instead of being buried inside the general Docker stack.

This reduces unnecessary dependencies between DNS and unrelated applications.

### Lesson

> **Critical infrastructure services should have simple and predictable dependency chains.**

---

# 6. Monitoring Should Be Layered

No single monitoring application provides complete observability.

The HomeLab eventually developed several complementary monitoring layers.

| Tool | Primary Role |
|---|---|
| Prometheus | Metrics collection |
| Grafana | Visualization and historical analysis |
| Uptime Kuma | Service availability |
| Netdata | Detailed real-time system monitoring |
| Glances | Lightweight host visibility |
| What's Up Docker | Container image update visibility |
| Speedtest Tracker | WAN performance history |
| Homepage | Central operational overview |

This initially looks like overlap, but the tools answer different questions.

For example:

```text
Is the service reachable?
        │
        ▼
   Uptime Kuma

Why is the host slow?
        │
        ▼
 Netdata / Glances

What changed over time?
        │
        ▼
Prometheus + Grafana

Are container images outdated?
        │
        ▼
What's Up Docker
```

### Lesson

> **Observability works best as multiple complementary layers rather than one universal dashboard.**

---

# 7. Monitoring Is Not the Same as Security Monitoring

Operational monitoring can reveal unusual behavior, but it does not automatically detect malicious activity.

For example, Prometheus may reveal:

```text
CPU spike
Memory growth
Network increase
Disk exhaustion
```

but cannot necessarily determine whether the cause is:

```text
Normal workload
Software bug
Misconfiguration
Attack
```

Security monitoring requires additional telemetry such as:

- Authentication events
- DNS logs
- Firewall logs
- Reverse-proxy logs
- Host security events
- IDS alerts

This led to future plans involving:

- Wazuh
- Suricata
- Centralized logging

### Lesson

> **Operational observability and security detection overlap, but they are not interchangeable.**

---

# 8. Update Visibility Is Better Than Blind Automation

Automatically updating every container can appear attractive.

However, unattended updates can introduce:

- Breaking changes
- Configuration incompatibilities
- Database migrations
- Unexpected downtime

The current approach favors update visibility through What's Up Docker while retaining control over when important services are upgraded.

A safer workflow is:

```text
Update Available
       │
       ▼
Review Change
       │
       ▼
Backup / Snapshot
       │
       ▼
Apply Update
       │
       ▼
Validate Service
```

### Lesson

> **Automation should reduce repetitive work without removing operational awareness.**

---

# 9. Snapshots Are Extremely Useful — but They Are Not Backups

Proxmox snapshots provide fast rollback before infrastructure changes.

They are particularly useful before:

- Package upgrades
- Configuration changes
- Application upgrades
- Infrastructure experiments

The workflow is simple:

```text
Snapshot
   │
   ▼
Change
   │
 ┌─┴─┐
 │   │
Good Bad
 │   │
 ▼   ▼
Keep Rollback
```

However, snapshots remain dependent on the underlying storage.

If that storage fails, the snapshot may disappear with it.

### Lesson

> **Snapshots protect against changes. Backups protect against loss.**

Both are required.

---

# 10. A Backup Must Have an Independent Failure Domain

The current external 3 TB HDD contains both:

```text
Media
+
Proxmox Backups
```

This is convenient, but it creates a shared failure domain.

If the disk physically fails:

```text
3 TB HDD Failure
       │
       ├── Media Lost
       │
       └── Local PVE Backups Lost
```

The backup therefore protects against certain workload failures, but not against failure of the backup disk itself.

Home Assistant improves this situation by maintaining an off-host backup copy in Google Drive.

### Lesson

> **A backup stored beside the data it protects may still share the same failure scenario.**

The long-term design should separate primary and backup storage and move closer to the 3-2-1 principle.

---

# 11. Recovery Is Part of Infrastructure Engineering

One of the most significant learning experiences involved the external media disk.

During storage reconfiguration, filesystem metadata was damaged.

Recovery required experimenting with several tools:

```text
TestDisk
   │
   ▼
Partition recovery attempt

R-Studio
   │
   ▼
Filesystem investigation

PhotoRec
   │
   ▼
Raw file carving
```

Although much of the media data could be recovered, original filenames and directory structures were not always preserved.

The media library then had to be reconstructed.

This changed the way storage operations are approached.

### Lesson

> **Knowing how to deploy infrastructure is only half the job. Knowing how to recover it is equally important.**

---

# 12. Read-Only First Is a Valuable Storage Rule

The storage incident resulted in a simple operational rule:

> **When the state of a disk is uncertain, inspect before modifying.**

Useful inspection commands include:

```bash
lsblk
lsblk -f
blkid
fdisk -l
df -h
mount
```

Where appropriate, unknown storage should initially be mounted read-only.

```bash
mount -o ro /dev/sdX1 /mnt/test
```

Destructive commands such as:

```text
wipefs
mkfs
fdisk write
gdisk write
```

should only be executed after confirming:

```text
Correct device
Correct filesystem
Correct capacity
Correct partitions
Existing data understood
Backup available
```

### Lesson

> **Storage commands deserve a higher verification threshold than ordinary configuration changes.**

---

# 13. Secondary Storage Should Not Prevent the Host from Booting

Another storage issue involved the external HDD configuration in:

```text
/etc/fstab
```

When the external disk was unavailable, the system waited for the device during startup and interrupted the normal Proxmox boot process.

The mount strategy was changed to use:

```text
nofail
```

This allows Proxmox to continue booting even if the external media disk is unavailable.

The desired behavior is:

```text
Media Disk Failure
       │
       ▼
Media Services Degraded
       │
       X
Proxmox Host Failure
```

### Lesson

> **Failure of a non-critical dependency should degrade only the services that depend on it, not the entire infrastructure.**

---

# 14. Shared Storage Requires Consistent Paths

The media environment involves several applications operating on the same files.

Examples include:

```text
qBittorrent
Sonarr
Radarr
Bazarr
Jellyfin
Samba
```

If each application sees the same file through unrelated paths, imports and hardlinks can become difficult to manage.

Conceptually:

```text
Same File

Container A → /downloads/movie.mkv
Container B → /data/downloads/movie.mkv
Container C → /mnt/media/movie.mkv
```

can create unnecessary path-mapping complexity.

The environment therefore benefits from predictable shared paths and clearly defined mount points.

### Lesson

> **Storage architecture should be designed before application configuration.**

A clean path model simplifies every service built on top of it.

---

# 15. Application Responsibilities Should Remain Separate

The media stack became easier to understand once each service had a clear responsibility.

```text
Jellyseerr
   ↓
Requests

Sonarr / Radarr
   ↓
Media Management

Prowlarr
   ↓
Indexer Management

qBittorrent
   ↓
Downloading

Bazarr
   ↓
Subtitles

Jellyfin
   ↓
Playback
```

Trying to make one application perform multiple infrastructure roles increases coupling.

The same principle applies elsewhere.

For example:

```text
Jellyfin ≠ File Server
Pi-hole ≠ Firewall
Nginx Proxy Manager ≠ Firewall
Homepage ≠ Monitoring Platform
```

### Lesson

> **Clear service boundaries make systems easier to operate, secure, replace, and troubleshoot.**

---

# 16. Reverse Proxies Do Not Replace Network Security

Nginx Proxy Manager simplifies access to internal web applications and provides centralized routing.

However:

```text
Reverse Proxy
      ≠
Firewall
```

A reverse proxy can provide:

- Hostname routing
- TLS termination
- Certificate management
- Application-level access controls

but it does not automatically provide:

- Network segmentation
- Lateral movement prevention
- Host firewalling
- Intrusion detection

### Lesson

> **Convenient access architecture should not be confused with security architecture.**

---

# 17. Remote Access and Download VPNs Solve Different Problems

Two different VPN concepts exist in the HomeLab.

## Remote Access VPN

Tailscale and WireGuard allow trusted devices to access the HomeLab remotely.

```text
Remote Device
      │
      ▼
Tailscale / WireGuard
      │
      ▼
Home LAN
```

## Outbound Privacy VPN

A future Gluetun deployment would route selected download traffic outward through a VPN provider.

```text
qBittorrent
     │
     ▼
Gluetun
     │
     ▼
VPN Provider
     │
     ▼
Internet
```

These architectures solve completely different problems.

### Lesson

> **"VPN" describes a technology, not a single use case.**

Remote administration and outbound privacy should be designed independently.

---

# 18. Experimental Workloads Need Different Trust Assumptions

The WebApp LXC provides a dedicated environment for:

- Development
- Application testing
- Deployment experiments
- Security testing

Separating it into its own LXC is useful, but the current flat LAN means the workload still shares network reachability with trusted systems.

The target architecture therefore introduces a dedicated:

```text
Lab VLAN
```

with restricted access to:

```text
Management
Services
IoT
```

### Lesson

> **A workload intentionally used for experimentation should not receive the same trust as stable infrastructure.**

---

# 19. Local-First Smart Home Design Improves Resilience

Home Assistant demonstrated the benefits of keeping automation logic local.

Local integrations and Zigbee allow many smart-home functions to continue without depending entirely on external cloud services.

```text
Device
   │
   ▼
Local Network / Zigbee
   │
   ▼
Home Assistant
```

rather than:

```text
Device
   │
   ▼
Internet
   │
   ▼
Vendor Cloud
   │
   ▼
Internet
   │
   ▼
Home
```

### Lesson

> **Reducing unnecessary cloud dependencies improves latency, privacy, and resilience.**

---

# 20. DNS Filtering Is Defense-in-Depth, Not a Firewall

Pi-hole provides:

- DNS filtering
- Tracker blocking
- Domain visibility
- Some malicious-domain blocking

However, it does not inspect all network traffic and cannot prevent communication performed directly through IP addresses or alternative DNS mechanisms.

### Lesson

> **Security controls should be understood according to what they actually enforce.**

Pi-hole contributes to defense-in-depth but does not replace firewalling, segmentation, or IDS/IPS.

---

# 21. Troubleshooting Should Move Through the Stack

A recurring lesson across Home Assistant, Jellyfin, Docker, networking, and monitoring was the importance of troubleshooting from the infrastructure layer upward.

Instead of immediately changing application configuration:

```text
Physical Host
     │
     ▼
Hypervisor
     │
     ▼
VM / LXC
     │
     ▼
Network
     │
     ▼
Storage
     │
     ▼
Service
     │
     ▼
Application
     │
     ▼
Client
```

For example, a Jellyfin playback problem could originate from:

- Jellyfin itself
- Storage permissions
- Missing media mount
- GPU passthrough
- Network connectivity
- Client codec support

### Lesson

> **Systematic troubleshooting reduces random configuration changes and shortens root-cause analysis.**

---

# 22. Observability Changes Troubleshooting

Before monitoring was introduced, troubleshooting was primarily reactive.

```text
Something feels slow
        │
        ▼
Open terminal
        │
        ▼
Investigate
```

With historical metrics:

```text
Problem Reported
       │
       ▼
Grafana / Prometheus
       │
       ▼
What changed?
       │
       ▼
Correlate CPU / RAM / Disk / Network
```

Historical data makes it possible to understand what happened before the problem was noticed.

### Lesson

> **Monitoring turns troubleshooting from guesswork into evidence-based investigation.**

---

# 23. Documentation Is Part of the Infrastructure

As the HomeLab grew, remembering every configuration decision became unrealistic.

Documentation now records:

- Architecture
- Network design
- Monitoring
- Cybersecurity
- Docker services
- Home Assistant
- Jellyfin
- Backup and recovery
- Lab design
- Lessons learned

Draw.io diagrams provide another abstraction layer.

```text
Infrastructure
      │
      ▼
Configuration
      │
      ▼
Documentation
      │
      ▼
Repeatability
```

### Lesson

> **Infrastructure that only exists in the administrator's memory is difficult to maintain and difficult to transfer.**

Documentation is therefore treated as part of the system itself.

---

# 24. Document the Current State and the Target State Separately

One of the most important documentation principles used throughout this repository is distinguishing between:

```text
CURRENT
```

and:

```text
TARGET / PLANNED
```

For example, the current network is primarily flat.

VLAN segmentation is planned.

Similarly:

- Gluetun is planned.
- Wazuh is planned.
- Suricata is planned.
- Dedicated backup storage is planned.
- Stronger Lab isolation is planned.

Presenting planned improvements as already implemented would make the documentation less technically credible.

### Lesson

> **Good technical documentation describes reality first and aspirations second.**

---

# 25. Security Is Usually a Series of Trade-Offs

Several HomeLab decisions involve compromises.

Examples include:

| Decision | Benefit | Trade-Off |
|---|---|---|
| Docker inside LXC | Efficient workload separation | Relaxed AppArmor |
| exFAT media disk | Windows/Linux portability | Limited Unix permissions |
| Flat LAN | Simple networking | Weak network isolation |
| Shared backup/media disk | Low cost and simplicity | Shared failure domain |
| LXC lab | Lightweight and recoverable | Shared host kernel |
| Local services | Control and privacy | More maintenance responsibility |

The objective is not always to eliminate every trade-off.

Instead:

```text
Identify
   ↓
Understand
   ↓
Document
   ↓
Mitigate
   ↓
Improve when justified
```

### Lesson

> **Engineering decisions should be evaluated by their risks, constraints, and operational consequences rather than by whether they are theoretically perfect.**

---

# 26. Small Infrastructure Can Teach Enterprise Concepts

Although this HomeLab runs on a single compact Proxmox host, many of the underlying concepts also appear in larger environments.

The project provides practical exposure to:

```text
Virtualization
Containerization
Network architecture
DNS
Reverse proxies
Monitoring
Metrics
Logging
Backup
Disaster recovery
Identity and access
Least privilege
Segmentation
Threat modeling
Change management
Documentation
```

The scale is smaller, but many of the engineering questions are the same.

For example:

```text
What happens if this service fails?

What systems trust this workload?

Where are the backups?

Who can access this interface?

How will I detect a problem?

How do I roll back this change?

What happens if this disk disappears?

Can this system be rebuilt?
```

These questions are more important than the number of servers involved.

---

# How the Architecture Evolved

The project gradually moved from:

```text
Install Services
      │
      ▼
Make Them Work
```

toward:

```text
Design
   │
   ▼
Deploy
   │
   ▼
Secure
   │
   ▼
Monitor
   │
   ▼
Back Up
   │
   ▼
Document
   │
   ▼
Improve
```

This change in approach is one of the most important outcomes of the project.

The HomeLab is no longer simply a collection of self-hosted applications.

It has become an environment for practicing infrastructure engineering.

---

# What I Would Design Differently Today

If rebuilding the environment from the beginning, several decisions would be made earlier.

## Network Segmentation

VLAN design would be considered before deploying large numbers of devices and services.

## Storage Architecture

Primary media storage and backup storage would use independent devices from the beginning.

## Path Planning

Shared Docker and media paths would be standardized before deploying the media automation stack.

## Monitoring

Prometheus and Grafana would be introduced earlier so historical data existed during initial troubleshooting.

## Lab Isolation

Experimental workloads would begin inside a dedicated network zone.

## Configuration Management

Reusable configurations would be version-controlled earlier.

## Recovery Testing

Restore procedures would be tested alongside backup configuration rather than later.

---

# Future Learning Goals

The next stages of the HomeLab can extend these lessons into additional areas.

Planned areas include:

- VLAN implementation
- Inter-VLAN firewalling
- Centralized logging
- Wazuh
- Suricata
- Proxmox Backup Server
- Independent backup storage
- Restore testing
- Ansible
- Infrastructure-as-Code
- Secret management
- Docker vulnerability scanning
- CI/CD experimentation
- Expanded security lab
- Additional Home Assistant automation
- Improved observability and alerting

These improvements are intended to build on the existing architecture rather than replace it without understanding why.

---

# Final Takeaway

The most valuable result of the HomeLab is not any individual application.

It is the process of repeatedly moving through:

```text
Problem
   │
   ▼
Investigation
   │
   ▼
Implementation
   │
   ▼
Failure / Testing
   │
   ▼
Understanding
   │
   ▼
Improved Design
   │
   ▼
Documentation
```

The project demonstrates that operating infrastructure involves more than getting services to run.

It requires understanding:

- Dependencies
- Failure domains
- Security boundaries
- Recovery mechanisms
- Observability
- Operational trade-offs

The HomeLab continues to evolve as those lessons are applied to each new iteration.

---

# Related Documentation

- [Architecture](architecture.md)
- [Networking](networking.md)
- [Monitoring](monitoring.md)
- [Cybersecurity](cybersecurity.md)
- [Backup & Recovery](backup-recovery.md)
- [Docker & Media Stack](docker-media-stack.md)
- [Home Assistant](home-assistant.md)
- [Jellyfin](jellyfin.md)
- [Lab Environment](lab-environment.md)
