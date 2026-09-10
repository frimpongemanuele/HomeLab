# Cybersecurity Design

## Overview

Security is a core design consideration of this HomeLab.

The environment hosts virtualization infrastructure, smart-home services, media applications, Docker workloads, DNS services, reverse-proxy infrastructure, file sharing, monitoring platforms, remote-access components, and a dedicated web-application testing environment.

The goal is not to reproduce a full enterprise security architecture inside a residential network, but to apply realistic cybersecurity principles such as:

- Reducing unnecessary exposure
- Isolating workloads
- Protecting management interfaces
- Using authenticated remote access
- Applying least privilege where possible
- Avoiding legacy protocols
- Monitoring infrastructure health
- Protecting recoverability through backups and snapshots
- Identifying trust boundaries and attack surfaces
- Documenting security trade-offs
- Gradually introducing stronger segmentation and detection capabilities

The current security model is intentionally pragmatic and continuously evolving.

---

## Security Objectives

The main security goals of the HomeLab are:

1. Keep infrastructure management interfaces private.
2. Avoid exposing administrative services directly to the public Internet.
3. Use encrypted VPN-based remote access.
4. Separate major workloads using VMs, LXC containers, and Docker containers.
5. Use authenticated access for shared resources.
6. Reduce dependency on obsolete or insecure protocols.
7. Protect critical workloads with snapshots and backups.
8. Minimize privileges assigned to service integrations.
9. Monitor infrastructure availability and abnormal resource behavior.
10. Separate experimental workloads from stable infrastructure.
11. Prepare the network for future VLAN-based segmentation.
12. Build a realistic platform for defensive-security experimentation.

---

# Current Security Architecture

The HomeLab currently follows a **low-exposure, defense-in-depth model**.

```text
                    Internet
                        │
                        ▼
              No Public Admin Ports
                        │
              ┌─────────┴─────────┐
              │                   │
         Tailscale            WireGuard
              │                   │
              └─────────┬─────────┘
                        │
                        ▼
                    Home LAN
                        │
                        ▼
                  Proxmox Host
                        │
        ┌───────────────┼────────────────┐
        │               │                │
   Management        Services        Smart Home
        │               │                │
    Proxmox UI        Jellyfin           HAOS
    Homepage          Docker
    Monitoring        Media Stack
        │
        ├── Pi-hole
        ├── Reverse Proxy
        └── WebApp Lab
```

Remote administrative access is performed through encrypted VPN connectivity instead of direct public exposure.

The current environment is primarily segmented through virtualization and service boundaries rather than dedicated network VLANs.

---

# Implemented Security Controls

## 1. VPN-Based Remote Access

Remote access is provided through:

- **Tailscale**
- **WireGuard**

Tailscale provides secure access to internal infrastructure without requiring inbound port forwarding.

WireGuard is also available through the Home Assistant environment for remote connectivity.

This reduces exposure compared with directly publishing services such as:

```text
Proxmox        :8006
Home Assistant :8123
Jellyfin       :8096
SSH            :22
```

to the public Internet.

### Security Benefits

- Administrative login pages are not intentionally reachable from arbitrary Internet hosts.
- Automated Internet scanners cannot directly enumerate most internal services.
- Remote traffic is encrypted.
- Public attack surface is reduced.
- Remote-access control is separated from individual application authentication.

---

## 2. Private Management Interfaces

Management interfaces remain on the private network or are accessed through the VPN layer.

Examples include:

- Proxmox administration
- Home Assistant administration
- Homepage
- Grafana
- Prometheus
- Pi-hole administration
- Nginx Proxy Manager
- Docker application interfaces
- Jellyfin administration
- qBittorrent Web UI

The design principle is:

```text
Internet
   │
   X
Direct Management Access
   │
   ▼
VPN Authentication
   │
   ▼
Internal Management Interface
```

This reduces unnecessary public exposure of administrative services.

---

## 3. Workload Isolation

Different workloads are separated using dedicated virtualization environments.

Current structure:

| Workload | Isolation |
|---|---|
| Home Assistant | Dedicated VM |
| Jellyfin | Dedicated LXC |
| Docker workloads | Dedicated LXC |
| Tailscale | Dedicated LXC |
| Homepage | Dedicated LXC |
| Pi-hole | Dedicated LXC |
| WebApp Lab | Dedicated LXC |

This provides useful isolation between different service categories.

For example, compromise of an experimental web application does not automatically provide direct filesystem access to the Home Assistant VM or Jellyfin container.

However, LXC containers share the Proxmox host kernel.

Therefore:

> **LXC isolation is useful, but it is not equivalent to the stronger kernel isolation provided by a full virtual machine.**

This distinction is important when evaluating security boundaries.

---

## 4. Dedicated WebApp Testing Environment

LXC 106 is reserved for web-application development and security testing.

This environment is considered higher-risk because workloads may be:

- Under development
- Intentionally misconfigured
- Running experimental dependencies
- Temporarily insecure
- Used for vulnerability testing
- Exposed to security-testing tools

Separating this workload from stable services reduces the risk of testing activity affecting production-like infrastructure.

A future improvement is to place this environment inside a dedicated **Lab VLAN** with restricted access to management and service networks.

---

## 5. Proxmox API Authentication

Dashboard and monitoring integrations use API-based access rather than relying on the primary interactive administrator password.

Conceptually:

```text
Homepage / Exporter
        │
        ▼
    API Token
        │
        ▼
   Proxmox API
```

API tokens provide better separation between interactive administrator access and application integrations.

The preferred model is:

- Dedicated service account
- Dedicated API token
- Read-only role where possible
- Minimum required privileges
- Independent revocation and rotation

This follows the principle of least privilege.

---

## 6. DNS Filtering with Pi-hole

Pi-hole provides centralized DNS-level filtering.

Its primary purpose is advertisement and tracker blocking, but it also contributes to security by:

- Blocking selected known malicious domains
- Reducing unwanted outbound requests
- Providing DNS query visibility
- Helping identify unusual DNS behavior
- Allowing centralized domain policy

Conceptually:

```text
Client
   │
   ▼
Pi-hole
   │
   ├── Blocked Domain ──► DENY
   │
   └── Allowed Domain
           │
           ▼
      Upstream DNS
```

Pi-hole is not treated as a replacement for:

- Firewalls
- IDS/IPS
- Endpoint protection
- Network segmentation

Instead, it provides another layer within the overall defense-in-depth model.

---

## 7. Authenticated Samba Access

The media storage is shared to Windows systems through Samba.

Anonymous guest access was replaced with a dedicated Samba user.

Example:

```bash
adduser mediauser
smbpasswd -a mediauser
```

The share uses authenticated access:

```ini
valid users = mediauser
```

This prevents unauthenticated clients from accessing the share.

---

## 8. SMB1 Disabled

SMB1 was briefly considered during troubleshooting but was not required.

The final design avoids SMB1 and relies on modern SMB versions.

This removes dependence on a legacy protocol with poor security properties and a long history of serious vulnerabilities.

---

## 9. Reverse Proxy Layer

Nginx Proxy Manager provides reverse-proxy functionality for selected internal applications.

This allows services to be accessed through centralized routing rather than exposing every backend directly by IP and port.

Potential benefits include:

- Centralized TLS termination
- Consistent internal hostnames
- Certificate management
- Reduced direct interaction with application ports
- Additional access-control opportunities

However:

> **A reverse proxy is not a firewall and does not automatically secure the backend application.**

Backend services still require authentication, patching, and appropriate network access controls.

---

## 10. Snapshot-Based Change Protection

Before significant infrastructure changes, snapshots are used where appropriate.

```text
Create Snapshot
      │
      ▼
Apply Change / Update
      │
      ▼
Validate Services
      │
   ┌──┴──┐
   │     │
Success Failure
   │     │
   ▼     ▼
Remove Rollback
Snapshot
```

Snapshots are not backups, but they provide fast recovery from:

- Failed upgrades
- Broken configuration changes
- Service regressions
- Administrative mistakes

---

## 11. Backup Strategy

The HomeLab currently uses multiple backup mechanisms.

### Home Assistant

Home Assistant backups are copied to Google Drive.

This provides an off-site recovery location.

### Proxmox

VMs and LXC containers are backed up to:

```text
/mnt/media/proxmox-backups
```

### Security Benefit

Backups improve resilience against:

- Configuration mistakes
- Failed updates
- Container corruption
- Accidental deletion
- Some ransomware scenarios
- Storage incidents

### Current Limitation

Media files and Proxmox backups currently reside on the same physical 3 TB HDD.

This creates a shared failure domain.

A future improvement is to move infrastructure backups to independent storage.

---

## 12. Monitoring and Observability

The HomeLab includes:

- Prometheus
- Grafana
- Uptime Kuma
- Netdata
- Glances
- What's Up Docker
- Speedtest Tracker
- Homepage

These tools primarily provide operational monitoring rather than full security detection.

They can still help identify abnormal conditions such as:

- Unexpected CPU spikes
- Sudden memory growth
- Excessive disk activity
- Unusual network utilization
- Repeated service failures
- Container restart loops
- Storage exhaustion

This telemetry provides useful context during troubleshooting and incident analysis.

More details are documented in:

[Monitoring & Observability](monitoring.md)

---

# Threat Model

The HomeLab is primarily designed to mitigate realistic threats relevant to self-hosted infrastructure.

Potential threat sources include:

- Internet-based scanning
- Vulnerable self-hosted applications
- Compromised IoT devices
- Malicious or compromised Docker containers
- Weak or leaked credentials
- Untrusted downloaded content
- Supply-chain vulnerabilities
- Misconfiguration
- Administrative mistakes
- Malware or ransomware on client systems
- Storage or hardware failure

The environment is not designed to resist a highly resourced targeted attacker.

The goal is to reduce common attack paths, limit impact, and improve recovery capability.

---

# Threat Scenarios

| Threat | Example | Current Mitigation |
|---|---|---|
| Internet scanning | Scanner searches for Proxmox or HA | No intentional public admin exposure |
| Credential attack | Brute-force attempt against management UI | VPN-only remote administration |
| Vulnerable web app | Exploitable test application | Dedicated WebApp LXC |
| Compromised Docker service | Container vulnerability | Dedicated Docker LXC |
| Compromised IoT device | Lateral movement attempt | Virtual workload separation; VLANs planned |
| DNS-based threat | Client resolves malicious domain | Pi-hole filtering |
| Legacy protocol attack | SMB1 exploitation | SMB1 disabled |
| Credential leakage | API token exposed | Separate API tokens and secrets excluded from Git |
| Failed update | Service becomes unavailable | Snapshot and rollback workflow |
| Storage failure | External HDD fails | Backups exist, but shared disk remains a limitation |
| Malware / ransomware | Writable SMB share targeted | Authenticated access; stronger segmentation planned |
| Supply-chain issue | Compromised container image | Controlled updates and trusted images |
| Administrative mistake | Wrong disk or config modified | Backup/recovery procedures and read-only-first policy |

---

# Attack Surface Analysis

## Proxmox VE

### Potential Risks

- Administrative interface compromise
- Weak administrator credentials
- Excessive API permissions
- SSH exposure
- Hypervisor vulnerabilities
- Compromise of the host affecting all workloads

### Current Mitigations

- Management interface kept private
- Remote access through VPN
- API-token integrations
- No intentional public exposure
- Backups and snapshots

### Planned Improvements

- Dedicated Management VLAN
- Restrict access to trusted administrator systems
- Stronger least-privilege API accounts
- Centralized authentication/security logging
- More granular firewall policy

---

# Home Assistant

Home Assistant is particularly sensitive because it controls physical devices and automation.

A compromise could potentially affect:

- Lights
- Switches
- Sensors
- Automation logic
- Zigbee devices
- Integrated smart-home systems

### Current Mitigations

- Dedicated VM
- Private management interface
- VPN-based remote access
- Independent Google Drive backups

### Planned Improvements

- Dedicated IoT VLAN
- Restrict IoT-to-management communication
- Explicit firewall rules
- Improved IoT monitoring

---

# Jellyfin

Potential attack surfaces include:

- Web interface
- User authentication
- Plugins
- Media parsing
- Downloaded media files
- GPU device access

### Current Mitigations

- Dedicated LXC
- No intentional direct public administration
- Media separated from system files
- GPU access limited to required `/dev/dri` devices

---

# Docker Host

The Docker LXC contains multiple independently maintained applications, including:

- Sonarr
- Radarr
- Prowlarr
- Bazarr
- qBittorrent
- Jellyseerr
- Dispatcharr
- Nginx Proxy Manager
- Prometheus
- Grafana
- Uptime Kuma
- Netdata
- Glances
- What's Up Docker
- Speedtest Tracker

This significantly increases application-level attack surface.

### Current Mitigation

Docker workloads run inside a dedicated LXC rather than directly on the Proxmox host.

### Security Trade-Off

Docker inside LXC requires additional capabilities.

Configuration includes:

```text
features: nesting=1,keyctl=1
lxc.apparmor.profile: unconfined
```

and some Docker containers may use:

```yaml
security_opt:
  - apparmor=unconfined
```

This reduces AppArmor confinement.

It solved Docker compatibility problems, but weakens one layer of isolation.

### Future Improvement

Higher-risk Docker workloads could be moved to a dedicated VM to provide stronger kernel isolation.

---

# qBittorrent

qBittorrent represents a higher-risk workload because it processes data from external peers.

Potential risks include:

- Malicious files
- Vulnerable Web UI
- Untrusted peer traffic
- Accidental exposure
- Excessive access to shared storage

The intended future design is:

```text
qBittorrent
     │
     ▼
Gluetun
     │
     ▼
Commercial VPN
     │
     ▼
Internet
```

Only downloader traffic should use the privacy VPN.

Management services such as Home Assistant, Proxmox, Jellyfin, and monitoring systems should not be routed through this tunnel.

---

# Nginx Proxy Manager

Potential risks include:

- Misconfigured proxy hosts
- Improper TLS configuration
- Administrative interface compromise
- Accidental exposure of internal services

### Current Mitigation

The service is used primarily for internal routing and is not considered a replacement for network access control.

### Planned Improvements

- Internal HTTPS expansion
- Certificate monitoring
- Stronger access restrictions
- Segmentation between proxy and management services

---

# Pi-hole

Potential risks include:

- Administrative interface compromise
- DNS manipulation
- Incorrect upstream configuration
- DNS service outage affecting clients

### Current Mitigations

- Dedicated LXC
- Separation from Docker
- Private network placement
- Centralized DNS management

---

# Samba

Potential risks include:

- Credential theft
- Malware modifying files
- Ransomware encrypting writable shares
- Overly permissive permissions

The exFAT media disk currently uses:

```text
umask=000
```

This provides very permissive filesystem access at the Linux level.

Access is therefore controlled mainly through Samba authentication rather than Unix filesystem permissions.

This is acceptable for the current portability-focused storage design, but is not ideal for strict least-privilege enforcement.

A dedicated NAS or Linux-native filesystem would allow more granular access control in the future.

---

# WebApp Lab

The WebApp LXC is intentionally treated as a higher-risk trust zone.

Potential risks include:

- Vulnerable frameworks
- Remote code execution
- Weak authentication
- Development interfaces
- Vulnerable dependencies
- SSRF
- File-upload vulnerabilities
- Command injection
- SQL injection
- Cross-site scripting

The environment can be used for controlled testing without deliberately weakening production-like services.

Future network segmentation should prevent this environment from freely accessing:

- Proxmox management
- Home Assistant
- Pi-hole administration
- Backup storage
- Other sensitive services

---

# Trust Boundaries

The HomeLab currently relies primarily on **logical trust boundaries** created through virtualization, containerization, application authentication, and VPN-based remote access.

![HomeLab Security Architecture - Current and Target State](../diagrams/exported/security-architecture.svg)

The diagram highlights the evolution of the security architecture.

### Current State

The current environment uses a primarily flat `192.168.1.0/24` LAN.

Although workloads are separated through VMs, LXC containers, and Docker containers, infrastructure, services, IoT devices, and lab workloads still share the same underlying network.

The main security boundaries currently come from:

- VPN-based remote access
- VM and LXC isolation
- Docker container isolation
- Application authentication
- Private management interfaces
- DNS filtering
- Service-level access controls

These controls reduce exposure, but they do not prevent all forms of lateral movement between systems on the LAN.

### Target State

The planned architecture introduces network-enforced trust boundaries using VLANs and inter-VLAN firewall policies.

The target security zones are:

- **Management** — Proxmox, administrative interfaces, and infrastructure management
- **Services** — Jellyfin, Docker applications, dashboards, and internal services
- **IoT** — smart-home and other embedded devices
- **Lab** — experimental and security-testing workloads
- **Guest** — untrusted or temporary client devices

Traffic between these zones will be explicitly controlled by firewall policy rather than implicitly trusted.

This represents the transition from:

```text
Logical workload isolation
          ↓
Network-enforced segmentation
          ↓
Least-privilege communication between zones
```

---

# Current Limitation: Flat LAN

The HomeLab still primarily operates on:

```text
192.168.1.0/24
```

This means systems with different trust levels can share the same Layer-2 network.

Examples include:

- Infrastructure management
- User devices
- IoT devices
- Media services
- Experimental workloads

Virtualization isolation does not automatically provide network isolation.

This is why VLAN segmentation is one of the highest-priority future improvements.

---

# Planned Network Segmentation

The target architecture introduces five trust zones:

```text
Management VLAN
Services VLAN
IoT VLAN
Lab VLAN
Guest VLAN
```

A future topology could look like:

```text
                     Router / Firewall
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
  Management VLAN     Services VLAN       IoT VLAN
        │                  │                  │
    Proxmox             Jellyfin          Smart Home
    Monitoring          Docker Apps       IoT Devices

        ┌──────────────────┴──────────────────┐
        │                                     │
        ▼                                     ▼
     Lab VLAN                            Guest VLAN
        │                                     │
   WebApp Testing                        Guest Devices
```

---

# Example Firewall Policy

The target policy follows a default-deny model between zones.

| Source | Destination | Policy |
|---|---|---|
| Management | Internal services | Allow required administration |
| Services | Management | Deny by default |
| IoT | Management | Deny |
| IoT | Home Assistant | Allow required protocols |
| IoT | Internet | Allow required outbound |
| Lab | Management | Deny |
| Lab | Services | Deny by default |
| Guest | Internal networks | Deny |
| Guest | Internet | Allow |
| VPN | Management | Allow authenticated administration |

This would significantly reduce lateral movement opportunities.

---

# Secrets Management

Sensitive values must not be committed to the public repository.

Examples include:

```text
Passwords
API tokens
Tailscale keys
WireGuard private keys
Jellyfin API keys
Home Assistant tokens
Samba passwords
VPN credentials
Cloud API credentials
```

Public configuration examples should use placeholders.

Example:

```env
PROXMOX_API_TOKEN=REDACTED
JELLYFIN_API_KEY=REDACTED
HOME_ASSISTANT_TOKEN=REDACTED
TAILSCALE_AUTH_KEY=REDACTED
```

Real `.env` files should be excluded through `.gitignore`.

Future improvements may include:

- SOPS
- Ansible Vault
- Docker secrets
- Dedicated secret-management platforms

---

# Container Supply-Chain Security

Docker introduces software supply-chain considerations.

Potential risks include:

- Compromised upstream images
- Outdated base images
- Vulnerable dependencies
- Malicious image updates

The current approach includes:

- Using trusted or well-maintained images
- Monitoring updates through What's Up Docker
- Reviewing updates before deployment
- Avoiding blind automatic updates
- Keeping persistent configuration separate from container images

Future improvements may include container image vulnerability scanning.

---

# Monitoring vs Security Monitoring

The current observability stack provides:

- Metrics
- Availability monitoring
- Resource trends
- Network-performance visibility
- Container-update visibility

These capabilities support security investigations, but they are not a SIEM or IDS.

Security monitoring requires additional telemetry such as:

```text
Authentication logs
Firewall logs
DNS logs
Reverse proxy logs
Linux audit events
Application logs
IDS alerts
```

The distinction is deliberate:

> **Operational monitoring helps identify abnormal behavior; security monitoring helps identify malicious behavior.**

---

# Planned Security Monitoring

## Wazuh

Potential use cases include:

- Host intrusion detection
- File-integrity monitoring
- Log collection
- Security event correlation
- Vulnerability information
- Centralized alerting

## Suricata

Potential use cases include:

- Network intrusion detection
- Traffic inspection
- Signature-based detection
- Network telemetry

## Centralized Logging

Possible platforms include:

- Grafana Loki
- Graylog
- Elastic Stack

Potential log sources include:

```text
Proxmox
Linux authentication
Docker
Tailscale
Pi-hole
Home Assistant
Samba
Nginx Proxy Manager
Firewall
```

---

# Defense-in-Depth Strategy

The long-term security model uses multiple layers.

```text
Layer 1
No unnecessary Internet exposure

Layer 2
VPN-based remote access

Layer 3
Network segmentation

Layer 4
Firewall policy

Layer 5
VM / LXC / Docker isolation

Layer 6
Application authentication

Layer 7
Least-privilege service accounts

Layer 8
DNS filtering

Layer 9
Monitoring and logging

Layer 10
Snapshots and backups

Layer 11
Recovery procedures
```

No individual control is considered sufficient on its own.

The objective is to:

- Reduce initial compromise opportunities
- Limit lateral movement
- Reduce privilege escalation paths
- Improve visibility
- Preserve recovery options

---

# Security Roadmap

## Phase 1 — Network Segmentation

- Management VLAN
- Services VLAN
- IoT VLAN
- Lab VLAN
- Guest VLAN
- Inter-VLAN firewall rules

## Phase 2 — Access Hardening

- Dedicated service accounts
- Least-privilege API tokens
- Restrict Proxmox management access
- Review SSH access
- Internal HTTPS expansion
- Certificate monitoring

## Phase 3 — Monitoring and Detection

- Centralized logging
- Wazuh
- Suricata
- Security dashboards
- Alerting
- Authentication-event monitoring

## Phase 4 — Storage and Backup Hardening

- Separate backup storage
- Encrypted off-site backups
- Restore testing
- Automated configuration backups
- Reduced writable exposure

## Phase 5 — Application Security

- WebApp lab expansion
- OWASP Top 10 testing
- Controlled vulnerability scanning
- Reverse-proxy hardening
- Docker image scanning
- Container capability review

## Phase 6 — Infrastructure Automation

- Ansible deployment
- Version-controlled configuration
- Secret management
- Automated compliance checks

---

# Security Lessons Learned

## Minimize Exposure

A service that does not need to be publicly reachable should remain private.

VPN access provides a safer administrative path than unnecessary port forwarding.

---

## Isolation Has Different Levels

VMs, LXCs, and Docker containers do not provide equivalent security boundaries.

Understanding the kernel boundary is critical when evaluating isolation.

---

## Compatibility Changes Can Reduce Security

Relaxing AppArmor solved Docker compatibility issues, but reduced confinement.

Security trade-offs should be explicitly documented.

---

## Experimental Workloads Need Their Own Trust Zone

The WebApp environment reinforces the need to separate development and testing workloads from stable services.

---

## DNS Can Provide Useful Security Visibility

DNS activity can reveal unexpected outbound communication even without full packet inspection.

---

## Backups Need Independent Failure Domains

A backup stored on the same physical disk as production data does not protect against disk failure.

---

## Recovery Is Part of Security

The storage incident demonstrated that resilience is not only about malicious threats.

Operational mistakes can have consequences similar to security incidents.

---

## Read-Only First

Before modifying important storage:

```bash
lsblk
lsblk -f
blkid
fdisk -l
```

and, where possible, mount the target filesystem read-only.

Destructive commands such as:

```text
wipefs
mkfs
fdisk write
gdisk write
```

should only be used after confirming the target disk and verifying that existing data is disposable.

---

# Current Security Limitations

The current environment still has several known limitations:

- Flat LAN
- No inter-VLAN firewalling yet
- Single Proxmox host
- Shared media and backup disk
- Docker inside LXC with relaxed AppArmor
- Permissive exFAT filesystem permissions
- No SIEM yet
- No IDS/IPS yet
- Limited centralized logging
- Experimental workloads still share the main LAN

These limitations are intentionally documented because they define the next stage of the HomeLab security roadmap.

---

# Key Takeaways

The current HomeLab security architecture demonstrates practical experience with:

- Attack-surface reduction
- VPN-based access
- Workload isolation
- Hypervisor security considerations
- LXC and Docker security trade-offs
- DNS filtering
- Reverse-proxy security
- API-token management
- SMB hardening
- Secrets hygiene
- Threat modeling
- Trust boundaries
- Network segmentation planning
- Backup and recovery
- Security monitoring design
- Web-application security experimentation

The environment is not presented as fully hardened or enterprise-grade.

Instead, it is documented as an evolving infrastructure platform where security controls can be designed, implemented, tested, monitored, and improved over time.

---

## Related Documentation

- [Architecture](architecture.md)
- [Networking](networking.md)
- [Monitoring & Observability](monitoring.md)
- [Docker & Media Stack](docker-media-stack.md)
- [Home Assistant](home-assistant.md)
- [Lab Environment](lab-environment.md)
- [Backup & Recovery](backup-recovery.md)
