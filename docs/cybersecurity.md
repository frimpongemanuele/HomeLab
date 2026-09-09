# Cybersecurity Design

## Overview

Security is a core design consideration of this HomeLab.

The environment hosts multiple internal services, including virtualization infrastructure, smart home automation, media services, Docker workloads, file sharing, monitoring, and remote-access components.

The objective is not to reproduce an enterprise security architecture in a residential environment, but to apply realistic cybersecurity principles such as:

* Reducing unnecessary exposure
* Isolating workloads
* Using authenticated remote access
* Limiting reliance on public-facing services
* Protecting administrative interfaces
* Maintaining backup and rollback capabilities
* Identifying attack surfaces and trust boundaries
* Documenting security trade-offs
* Gradually introducing segmentation, monitoring, and detection capabilities

The current model is intentionally pragmatic: security controls are introduced where they provide meaningful value without making the environment unnecessarily complex.

---

## Security Objectives

The main security goals of the HomeLab are:

1. Keep infrastructure management interfaces private.
2. Avoid exposing administrative services directly to the public Internet.
3. Use encrypted VPN-based remote access.
4. Separate major workloads using VMs and containers.
5. Use authenticated access for shared resources.
6. Reduce dependency on legacy or insecure protocols.
7. Protect critical workloads with snapshots and backups.
8. Maintain visibility into system availability and resource usage.
9. Provide a platform for future security monitoring and network-security experimentation.

---

# Current Security Architecture

The HomeLab currently follows a **low-exposure architecture**.

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
          ┌─────────────┼─────────────┐
          │             │             │
     Management      Services      Smart Home
          │             │             │
      Proxmox UI      Jellyfin        HAOS
      Homepage        Docker
                      Media Stack
```

Remote administrative access is performed through encrypted VPN connectivity rather than exposing internal management interfaces through conventional port forwarding.

---

# Implemented Security Controls

## 1. VPN-Based Remote Access

Remote access is provided through:

* **Tailscale**
* **WireGuard**

Tailscale provides secure access to internal infrastructure without requiring inbound port-forwarding rules on the home router.

WireGuard is also available through the Home Assistant environment for remote connectivity.

This approach significantly reduces exposure compared with directly publishing services such as:

```text
Proxmox    :8006
Home Assistant :8123
Jellyfin   :8096
SSH        :22
```

to the Internet.

### Security Benefit

Without public port forwarding:

* Internet-wide scanners cannot directly enumerate most internal services.
* Administrative login pages remain inaccessible from arbitrary Internet hosts.
* The attack surface exposed at the network perimeter is reduced.
* Remote traffic is encrypted before reaching the internal environment.

---

## 2. Private Management Interfaces

Infrastructure administration is performed from the trusted LAN or through the VPN.

Examples include:

* Proxmox administration interface
* Home Assistant administration
* Homepage dashboard
* Docker application interfaces
* Jellyfin administration
* qBittorrent Web UI

These services are not intentionally exposed directly to the public Internet.

The design principle is:

```text
Internet
   │
   ✕
Direct Management Access
   │
   ▼
VPN Authentication
   │
   ▼
Internal Service
```

---

## 3. Workload Isolation

Different services are separated into dedicated virtualization environments.

Current structure:

| Workload         | Isolation     |
| ---------------- | ------------- |
| Home Assistant   | Dedicated VM  |
| Jellyfin         | Dedicated LXC |
| Docker workloads | Dedicated LXC |
| Tailscale        | Dedicated LXC |
| Homepage         | Dedicated LXC |

This provides a useful isolation boundary between services.

For example, compromise of a web application running inside the Docker host does not automatically mean that the attacker has direct access to the Home Assistant operating system or the Jellyfin container.

However, LXC containers share the Proxmox host kernel and therefore do not provide the same isolation boundary as a full virtual machine.

This distinction is important when evaluating container security.

---

## 4. Proxmox API Authentication

Homepage retrieves infrastructure information from Proxmox through an API token rather than storing the primary interactive administrator password.

Conceptually:

```text
Homepage
    │
    │ API Token
    ▼
Proxmox API
```

Using API tokens makes credentials easier to separate, rotate, and revoke.

### Current Security Consideration

The security of this integration depends on the permissions assigned to the token.

A future improvement is to ensure that dashboard integrations use:

* A dedicated service account
* A dedicated API token
* Read-only permissions
* The minimum required privileges

This follows the principle of least privilege.

---

## 5. Authenticated Samba Access

The media storage is exposed to Windows clients using Samba.

Anonymous guest access was replaced with an authenticated Samba user.

Example:

```bash
adduser mediauser
smbpasswd -a mediauser
```

The share requires:

```ini
valid users = mediauser
```

This prevents completely unauthenticated access to the media share.

---

## 6. SMB1 Disabled

SMB1 was briefly enabled during troubleshooting but was later identified as unnecessary and insecure.

The final design avoids SMB1 and relies on modern SMB versions.

This reduces exposure to a legacy protocol associated with well-known historical vulnerabilities and poor security properties.

---

## 7. Snapshot-Based Change Protection

Before significant infrastructure changes, snapshots are used where appropriate.

Workflow:

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

Snapshots are not backups, but they provide useful protection against configuration mistakes and failed upgrades.

---

## 8. Backup Strategy

The HomeLab currently uses multiple backup mechanisms.

### Home Assistant

Home Assistant backups are copied to Google Drive.

This provides an off-site recovery copy for one of the most important services in the environment.

### Proxmox

VMs and LXC containers are backed up to the external HDD.

Current location:

```text
/mnt/media/proxmox-backups
```

### Security Benefit

Backups provide resilience against:

* Configuration mistakes
* Failed updates
* VM/container corruption
* Accidental deletion
* Some forms of ransomware or destructive activity

However, the current local backup architecture has an important limitation:

> Media files and Proxmox backups currently reside on the same physical 3 TB HDD.

Therefore, physical disk failure would affect both datasets.

Separating the backup destination from the media storage is a planned improvement.

---

# Threat Model

The HomeLab is primarily exposed to threats originating from:

* The local network
* Compromised IoT devices
* Vulnerable self-hosted applications
* Misconfiguration
* Weak or leaked credentials
* Malicious or compromised Docker containers
* Supply-chain vulnerabilities
* Untrusted downloaded content
* Administrative mistakes
* Storage or hardware failure

The environment is not designed to defend against a highly resourced targeted attacker.

Instead, the security model focuses on realistic threats relevant to self-hosted infrastructure.

---

## Threat Scenarios

| Threat                 | Example                                      | Current Mitigation                                      |
| ---------------------- | -------------------------------------------- | ------------------------------------------------------- |
| Internet scanning      | Automated scanner searches for Proxmox or HA | No direct port forwarding                               |
| Credential attack      | Brute-force attempt against management UI    | VPN-only remote access                                  |
| Compromised service    | Vulnerable Docker application                | Workload isolation                                      |
| Compromised IoT device | IoT device attempts lateral movement         | Service separation; VLAN segmentation planned           |
| Legacy protocol attack | SMB1 exploitation                            | SMB1 disabled                                           |
| Credential leakage     | Dashboard integration token exposed          | API tokens used; secrets excluded from Git              |
| Failed update          | Service becomes unusable                     | Snapshot / rollback workflow                            |
| Storage failure        | External HDD becomes unavailable             | Backups exist, but local single-disk dependency remains |
| Malware/ransomware     | Writable SMB share targeted                  | Authenticated access; stronger isolation planned        |
| Container compromise   | Web app gains shell inside container         | Dedicated Docker LXC                                    |
| Administrative mistake | Wrong disk formatted or mount misconfigured  | Backup/recovery process and read-only-first lessons     |

---

# Attack Surface Analysis

## Proxmox VE

### Potential Risks

* Administrative interface compromise
* Weak credentials
* Excessive API token permissions
* SSH exposure
* Hypervisor vulnerabilities

### Current Mitigations

* Management interface kept internal
* Remote access through VPN
* API token support for integrations
* No intended public exposure

### Planned Improvements

* Dedicated management VLAN
* Restrict access to trusted administrator devices
* Least-privilege API accounts
* More centralized logging

---

## Home Assistant

Home Assistant is a particularly sensitive component because it interacts with physical devices.

A compromise could potentially affect:

* Lights
* Switches
* Sensors
* Automation logic
* Zigbee devices
* Other integrated smart-home systems

### Current Mitigations

* Dedicated virtual machine
* No unnecessary public exposure
* VPN-based access
* Independent backups to Google Drive

### Planned Improvements

* Place IoT devices into a dedicated VLAN
* Restrict IoT-to-LAN communication
* Explicit firewall rules between IoT and management networks

---

## Jellyfin

Potential attack surface includes:

* Web interface
* Media parsing
* Plugins
* User authentication
* Uploaded/downloaded media content

### Current Mitigations

* Dedicated LXC
* No direct administrative exposure
* Media data separated from Jellyfin system files
* Hardware access limited to required GPU devices

---

## Docker Host

The Docker LXC contains several independently maintained applications.

This increases the overall application attack surface.

Current workloads include:

* Radarr
* Sonarr
* Prowlarr
* Bazarr
* qBittorrent

A vulnerability in any of these services could provide an attacker with access to the Docker environment.

### Current Mitigation

Docker workloads are placed inside a dedicated LXC rather than directly on the Proxmox host.

### Important Security Trade-Off

Docker runs inside LXC with additional permissions required for nested containerization.

The configuration includes:

```text
features: nesting=1,keyctl=1
lxc.apparmor.profile: unconfined
```

and, for some Docker containers:

```yaml
security_opt:
  - apparmor=unconfined
```

This reduces AppArmor confinement.

It solved compatibility issues with Docker inside LXC, but it also weakens one layer of host-side isolation.

This is therefore documented as a conscious security trade-off rather than being treated as a neutral configuration change.

### Possible Future Improvement

If stronger isolation becomes a priority, the Docker environment could be migrated to:

* A dedicated VM

rather than Docker inside an LXC container.

That would provide a stronger kernel isolation boundary.

---

## qBittorrent

The download client represents a higher-risk workload because it processes data obtained from external peers.

Potential risks include:

* Malicious files
* Vulnerable Web UI
* Network exposure
* Untrusted peer traffic

The intended future design is:

```text
qBittorrent
     │
     ▼
Gluetun
     │
     ▼
Commercial VPN Provider
     │
     ▼
Internet
```

Only download traffic should use the privacy VPN.

Home Assistant, Proxmox, Jellyfin, and other internal services should not be routed through this tunnel.

---

## Samba

The Samba share provides read/write access to the shared media disk.

Potential risks include:

* Credential theft
* Malware modifying media
* Ransomware encrypting writable files
* Excessively permissive filesystem permissions

Current configuration uses authenticated access.

However, the exFAT filesystem requires simplified ownership semantics.

The current mount configuration uses:

```text
umask=000
```

which makes filesystem permissions very permissive at the Linux level.

Access is therefore largely controlled at the Samba/service layer rather than through traditional Unix file permissions.

This is acceptable for the current portability-focused design but is not ideal from a least-privilege perspective.

A future dedicated NAS or Linux-native storage system would allow more granular access control.

---

# Trust Boundaries

The HomeLab currently contains several logical trust boundaries.

```text
                   Internet
                      │
              ───── Trust Boundary ─────
                      │
                 VPN Layer
                      │
              ───── Trust Boundary ─────
                      │
                   Home LAN
                      │
          ┌───────────┴────────────┐
          │                        │
     Management                IoT / Clients
          │                        │
          ▼                        ▼
       Proxmox                 Smart Devices
          │
    ─── Virtualization Boundary ───
          │
     VM / LXC / Docker
```

At present, many of these boundaries are logical rather than enforced through dedicated VLANs.

Implementing network segmentation is therefore one of the main planned security improvements.

---

# Secrets Management

Sensitive values should never be committed to the public repository.

Examples include:

```text
Passwords
API tokens
Tailscale keys
WireGuard private keys
Jellyfin API keys
Home Assistant tokens
Samba passwords
Cloud API credentials
```

Configuration examples in the repository should use placeholders:

```env
PROXMOX_API_TOKEN=REDACTED
JELLYFIN_API_KEY=REDACTED
HOME_ASSISTANT_TOKEN=REDACTED
TAILSCALE_AUTH_KEY=REDACTED
```

Where possible, `.env` files containing real credentials should be excluded through `.gitignore`.

Future improvements may include:

* SOPS
* Ansible Vault
* Docker secrets
* Dedicated secret-management platforms

---

# Security Monitoring

The HomeLab includes or is being expanded with monitoring services such as:

* Prometheus
* Grafana
* Uptime Kuma
* Proxmox metrics
* Jellyfin metrics
* Home Assistant metrics

These primarily provide availability and performance monitoring.

They should not be confused with security monitoring.

The next stage is to introduce security-focused telemetry.

Potential tools include:

### Wazuh

Possible uses:

* Host intrusion detection
* File integrity monitoring
* Security event collection
* Vulnerability information
* Centralized alerts

### Suricata

Possible uses:

* Network intrusion detection
* Traffic inspection
* Signature-based detection
* Network telemetry

### Centralized Logging

Possible platforms:

* Grafana Loki
* Graylog
* Elastic Stack

Potential log sources:

```text
Proxmox
Linux authentication
Docker
Tailscale
Home Assistant
Samba
Reverse proxy
Firewall
```

---

# Planned Network Segmentation

The current HomeLab primarily operates on a single home LAN.

One of the most important future security improvements is VLAN-based segmentation.

The planned architecture is:

```text
                    Router / Firewall
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
      Management       Services        IoT
        VLAN              VLAN         VLAN
            │             │             │
        Proxmox         Jellyfin     Smart Devices
        Admin PC        Docker       Hue Bridge
        Homepage        Media        Zigbee Gateway

                          │
                          ▼
                     Guest VLAN
```

Possible VLAN roles:

| VLAN       | Purpose                                   |
| ---------- | ----------------------------------------- |
| Management | Proxmox and administrative interfaces     |
| Services   | Jellyfin, Docker applications, dashboards |
| IoT        | Smart-home devices                        |
| Guest      | Untrusted client devices                  |

---

## Example Firewall Policy

The future policy would follow a deny-by-default model between security zones.

Example:

```text
Admin PC → Management VLAN      ALLOW
Management → IoT                ALLOW only required flows
IoT → Management                DENY
Guest → Management              DENY
Guest → IoT                     DENY
IoT → Internet                  ALLOW where required
Internet → Management           DENY
VPN → Management                ALLOW authenticated devices
```

This would significantly reduce lateral movement opportunities if an IoT or guest device were compromised.

---

# Defense-in-Depth Strategy

The long-term security model uses multiple layers:

```text
Layer 1
No unnecessary Internet exposure

Layer 2
VPN authentication

Layer 3
Network segmentation

Layer 4
Firewall policy

Layer 5
VM / LXC / Docker isolation

Layer 6
Application authentication

Layer 7
Least-privilege credentials

Layer 8
Logging and monitoring

Layer 9
Snapshots and backups

Layer 10
Recovery procedures
```

No individual security control is assumed to be perfect.

The goal is to make compromise more difficult, reduce lateral movement, and improve the ability to detect and recover from incidents.

---

# Known Security Limitations

The current environment has several known limitations.

### Single LAN

Most devices currently share the same network.

This increases the potential for lateral movement.

**Planned mitigation:** VLAN segmentation.

---

### Single Proxmox Node

The entire virtualized environment depends on one physical host.

This is primarily an availability risk.

**Possible future mitigation:** secondary node or improved recovery automation.

---

### Shared Media and Backup Disk

Media and Proxmox backups currently reside on the same physical HDD.

This does not provide physical failure isolation.

**Planned mitigation:** dedicated backup storage.

---

### Docker AppArmor Relaxation

Docker inside LXC requires relaxed AppArmor restrictions.

**Possible future mitigation:** migrate Docker workloads to a VM.

---

### Permissive exFAT Permissions

The portable exFAT media drive does not provide Linux-native access-control semantics.

**Possible future mitigation:** dedicated NAS or Linux-native filesystem with controlled SMB exports.

---

### Limited Security Telemetry

Performance monitoring exists, but SIEM/IDS capabilities are not yet fully implemented.

**Planned mitigation:** Wazuh, Suricata, and centralized logging.

---

# Security Roadmap

Planned improvements are prioritized approximately as follows:

### Phase 1 — Network Segmentation

* Management VLAN
* Services VLAN
* IoT VLAN
* Guest VLAN
* Inter-VLAN firewall rules

### Phase 2 — Access Hardening

* Dedicated service accounts
* Least-privilege API tokens
* Restrict Proxmox management access
* Review SSH authentication
* Review administrative accounts

### Phase 3 — Monitoring and Detection

* Centralized log collection
* Wazuh deployment
* Suricata testing
* Security dashboards
* Alerting

### Phase 4 — Storage and Backup Hardening

* Separate backup disk
* Encrypted off-site backups
* Restore testing
* Automated configuration backups

### Phase 5 — Infrastructure Automation

* Ansible deployment
* Version-controlled configurations
* Secret management
* Automated compliance checks

---

# Security Lessons Learned

Several practical lessons emerged during the construction of this HomeLab.

### Minimize Exposure

A service that does not need to be publicly reachable should not be publicly reachable.

VPN-based access provides a practical alternative to exposing administrative interfaces.

### Isolation Has Levels

VMs, LXC containers, and Docker containers do not provide equivalent security boundaries.

Understanding where the kernel boundary exists is important when designing isolation.

### Compatibility Changes Can Reduce Security

Disabling AppArmor restrictions solved Docker compatibility problems, but it also reduced confinement.

Security trade-offs should be explicitly documented.

### Backups Must Have Independent Failure Domains

A backup stored on the same physical disk as the data it protects does not protect against disk failure.

### Recovery Is Part of Security

The storage incident demonstrated that security is not limited to preventing malicious activity.

Operational mistakes, data corruption, failed mounts, and incorrect disk operations can have consequences similar to a security incident.

Recovery procedures are therefore part of the overall resilience strategy.

### Read-Only First

Before modifying unknown or important storage devices:

```bash
lsblk
lsblk -f
blkid
fdisk -l
```

and, where possible, mount the filesystem read-only before performing destructive operations.

Destructive commands such as:

```text
wipefs
mkfs
fdisk write
gdisk write
```

should only be used after verifying the target device and confirming that the existing data is disposable.

---

# Conclusion

The HomeLab currently follows a pragmatic security model centered around:

* Minimal public exposure
* VPN-based remote access
* Workload separation
* Authenticated services
* Backup and rollback capabilities
* Awareness of infrastructure attack surfaces
* Explicit documentation of security trade-offs

The environment is not presented as fully hardened or enterprise-grade.

Instead, it serves as an evolving platform where security controls can be designed, implemented, tested, monitored, and improved over time.

The next major milestone is **network segmentation and security monitoring**, which will introduce stronger trust boundaries and provide the telemetry required for more advanced defensive-security experimentation.

-- monitoring
