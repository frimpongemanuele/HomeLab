# HomeLab Architecture

## Overview

This document describes the architecture of my self-hosted HomeLab environment.

The lab is built around a **Lenovo ThinkCentre M720q Tiny** running **Proxmox VE** and is designed as a modular platform for:

* Virtualization
* Containerization
* Smart home automation
* Media services
* Secure remote access
* Networking
* Monitoring and observability
* Web application testing
* Cybersecurity experimentation

The infrastructure evolved from an older laptop-based server into a dedicated mini-PC platform to improve reliability, isolation, scalability, and maintainability.

The current design follows several core principles:

* Separate major workloads using VMs, LXC containers, and Docker containers
* Keep management services private
* Avoid unnecessary public exposure
* Use centralized shared storage for media and backups
* Maintain portability of the external storage device
* Separate stable infrastructure from experimental workloads
* Add monitoring and observability as first-class infrastructure components
* Keep the environment simple enough to maintain while allowing future expansion

---

## High-Level Architecture

![HomeLab High-Level Architecture](../diagrams/exported/homelab-architecture.svg)

The Proxmox host acts as the central compute node of the environment.

At a high level, the infrastructure is organized into:

1. Physical and network layer
2. Virtualization layer
3. Application and service layer
4. Storage layer
5. Remote-access layer
6. Monitoring and observability layer
7. Experimental lab layer

---

# Physical Infrastructure

## Proxmox Host

The primary compute node is a:

**Lenovo ThinkCentre M720q Tiny**

Current hardware configuration:

| Component       | Specification          |
| --------------- | ---------------------- |
| CPU             | Intel Core i5-8500T    |
| Cores / Threads | 6 / 6                  |
| Base Clock      | 2.1 GHz                |
| Turbo Clock     | Up to 3.5 GHz          |
| Integrated GPU  | Intel UHD Graphics 630 |
| RAM             | 16 GB DDR4 SO-DIMM     |
| Primary Storage | 256 GB NVMe SSD        |
| Network         | Gigabit Ethernet       |
| Hypervisor      | Proxmox VE             |

The system was selected because of its relatively low power consumption, compact size, Intel virtualization support, and integrated GPU.

The **Intel UHD Graphics 630** is also used by Jellyfin for hardware-accelerated transcoding through **VAAPI**.

---

# Network Layer

The HomeLab currently operates inside the main home LAN.

The primary router is a **TIM HUB+**, which provides:

* Internet gateway
* DHCP
* Local DNS forwarding
* NAT
* Wi-Fi connectivity
* Gigabit Ethernet connectivity

The Proxmox host and core services use stable internal addresses through static configuration or DHCP reservations.

The current environment is primarily based on a single LAN, while VLAN segmentation is planned as a future improvement.

---

## Remote Access

Remote access is implemented through encrypted VPN technologies rather than exposing administrative services directly to the Internet.

The environment currently uses:

* **Tailscale**
* **WireGuard**

Tailscale is used as the main secure remote-access mechanism for infrastructure services.

WireGuard is also available through Home Assistant for remote connectivity.

This provides secure access to services such as:

* Proxmox
* Home Assistant
* Jellyfin
* Homepage
* Monitoring interfaces
* Internal application dashboards

without requiring conventional public port forwarding.

---

# Virtualization Layer

**Proxmox VE** provides the virtualization layer for the lab.

The environment intentionally combines:

* KVM virtual machines
* LXC system containers
* Docker application containers

Each technology is used according to the requirements of the workload.

---

## Virtual Machines

| ID  | Workload          | Type |
| --- | ----------------- | ---- |
| 100 | Home Assistant OS | VM   |

---

## LXC Containers

| ID  | Workload  | Role                                         |
| --- | --------- | -------------------------------------------- |
| 101 | Jellyfin  | Media server                                 |
| 102 | Docker    | Containerized application host               |
| 103 | Tailscale | Secure remote access                         |
| 104 | Homepage  | Centralized dashboard                        |
| 105 | Pi-hole   | DNS filtering and ad blocking                |
| 106 | WebApp    | Development and security testing environment |

The environment therefore separates infrastructure services, user-facing services, and experimental workloads at the virtualization layer.

---

# Home Assistant VM

Home Assistant OS runs as a dedicated virtual machine rather than inside a container.

This provides a complete Home Assistant appliance environment and simplifies:

* Supervisor and add-on management
* Integrations
* USB device passthrough
* Backup and restore operations
* Smart home device management
* Zigbee integration

The VM interfaces with Zigbee devices through a **SONOFF Zigbee 3.0 USB Dongle Plus**.

A **Philips Hue Bridge** is also part of the smart lighting environment.

Home Assistant backups are additionally stored off-site through Google Drive.

More details are documented in:

[Home Assistant](home-assistant.md)

---

# Jellyfin LXC

Jellyfin runs inside a dedicated LXC container.

Using LXC provides lower overhead than a full virtual machine while still separating the media server from the Proxmox host and Docker application stack.

The Jellyfin container receives access to:

```text
/dev/dri/card0
/dev/dri/renderD128
```

This enables VAAPI-based hardware transcoding using the integrated Intel UHD Graphics 630 GPU.

The media storage is exposed to the container through a Proxmox bind mount.

Inside Jellyfin, the media libraries are available under:

```text
/media/movies
/media/tv
```

This allows Jellyfin to remain independent from the media automation stack while still consuming the same shared library.

Further configuration is documented in:

[Jellyfin](jellyfin.md)

---

# Docker Application Layer

A dedicated LXC container is used as the Docker host.

This provides the following architecture:

```text
Physical Host
    │
    ▼
Proxmox VE
    │
    ▼
LXC 102
    │
    ▼
Docker Engine
    │
    ▼
Application Containers
```

Docker Compose is used to manage multiple services grouped by function.

---

## Media Automation Services

The media automation stack currently includes:

* Sonarr
* Radarr
* Prowlarr
* Bazarr
* qBittorrent
* Jellyseerr
* Dispatcharr

These services automate media requests, downloads, organization, subtitles, and IPTV/live TV management.

---

## Networking Services

Networking-related Docker services include:

* Nginx Proxy Manager
* Speedtest Tracker

Nginx Proxy Manager is used as a reverse proxy for selected internal services.

Speedtest Tracker records Internet connection performance over time.

---

## Monitoring and Maintenance Services

The Docker host also runs several monitoring and maintenance services:

* Prometheus
* Grafana
* Uptime Kuma
* Netdata
* Glances
* What's Up Docker

These provide metrics collection, dashboards, service-health monitoring, real-time resource visibility, and Docker image update tracking.

The Docker implementation is documented in:

[Docker & Media Stack](docker-media-stack.md)

---

# Pi-hole LXC

Pi-hole runs in a dedicated lightweight LXC container.

Its main functions include:

* DNS-level advertisement blocking
* Tracker filtering
* Centralized DNS policy
* DNS query visibility
* Blocking selected unwanted domains

Running Pi-hole independently from Docker improves resilience because DNS filtering remains available even if the Docker host is restarted or undergoing maintenance.

Pi-hole also provides useful network telemetry that can help identify unexpected DNS activity.

More detailed network documentation is available in:

[Networking](networking.md)

---

# Web Application Testing LXC

LXC 106 is reserved for web application development and testing.

This environment is intentionally separated from stable services such as Home Assistant, Jellyfin, and DNS infrastructure.

The WebApp LXC can be used for:

* Deploying experimental web applications
* Testing application frameworks
* Reverse proxy testing
* Web server administration
* Application hardening
* Controlled security testing
* Vulnerability assessment
* OWASP-focused experiments

Because test applications may be intentionally insecure or unstable, this container is considered a higher-risk workload.

A future improvement is to place this environment inside a dedicated **Lab VLAN** with restricted access to production-like services.

See:

[Lab Environment](lab-environment.md)

---

# Shared Storage Architecture

The lab uses an external **3 TB Seagate ST3000DM001 HDD** connected through an **ICY BOX USB enclosure**.

The drive is mounted on the Proxmox host at:

```text
/mnt/media
```

The current filesystem is:

```text
exFAT
```

exFAT was selected because the disk must remain portable and readable from:

* Linux / Proxmox
* Windows
* macOS
* Other computers if required

The current structure is approximately:

```text
/mnt/media/
├── movies/
├── tv/
└── proxmox-backups/
```

The same physical disk currently stores:

* Jellyfin media
* Proxmox backups

This is convenient for the current HomeLab, but it creates a **shared failure domain**.

If the physical disk fails, both the media library and local Proxmox backups are affected.

A future improvement is therefore to separate media storage from backup storage.

---

# Storage Mount Strategy

Because the external disk may not always be connected or available during boot, the Proxmox host must remain bootable without it.

The drive is mounted using UUID-based configuration and the `nofail` option.

Example:

```fstab
UUID=XXXX-XXXX /mnt/media exfat defaults,nofail,uid=1000,gid=1000,umask=000 0 0
```

The `nofail` option was introduced after an earlier storage configuration caused Proxmox to enter emergency mode when the external disk could not be mounted.

This incident directly influenced the current storage design.

---

# File Sharing

The shared media disk is also exposed to Windows systems through **Samba**.

Example path:

```text
\\PROXMOX-HOST\media
```

A dedicated Samba account is used instead of guest access.

SMB1 is disabled because it is obsolete and insecure.

This provides a convenient method for transferring and managing media files from Windows without disconnecting the physical HDD from the Proxmox host.

---

# Media Data Flow

The media automation workflow is organized into several stages.

```text
Jellyseerr
     │
     ▼
Sonarr / Radarr
     │
     ├──────► Prowlarr
     │
     ▼
qBittorrent
     │
     ▼
Downloads
     │
     ▼
Sonarr / Radarr Import
     │
     ├──────────────┐
     │              │
     ▼              ▼
/media/tv      /media/movies
     │              │
     └───────┬──────┘
             ▼
          Jellyfin
```

Bazarr provides subtitle automation alongside the media libraries.

Dispatcharr is used for IPTV and live TV management.

This design separates:

* Content requests
* Indexing
* Downloading
* Media organization
* Subtitle management
* Media playback

More detail is available in:

[Docker & Media Stack](docker-media-stack.md)

---

# Reverse Proxy Layer

**Nginx Proxy Manager** provides reverse-proxy functionality for selected internal web applications.

This allows internal services to be accessed using easier-to-manage hostnames rather than individual IP and port combinations.

The reverse proxy also provides a foundation for:

* Internal HTTPS
* Centralized TLS termination
* Consistent service hostnames
* Reduced direct interaction with application ports

Reverse proxying is treated as an access-management layer and not as a replacement for firewalling or network segmentation.

---

# Dashboard and Operational Overview

**Homepage** provides the main centralized dashboard for the HomeLab.

It aggregates information from multiple services and displays their operational state in one interface.

The dashboard is currently organized into areas such as:

* Smart Home
* Media
* Infrastructure
* Downloads
* Network
* Monitoring

Integrated services include:

* Home Assistant
* Jellyfin
* Proxmox
* Docker
* Tailscale
* Sonarr
* Radarr
* Prowlarr
* qBittorrent
* Bazarr
* Jellyseerr
* Dispatcharr
* Pi-hole
* Nginx Proxy Manager
* Speedtest Tracker
* Uptime Kuma
* Netdata
* Glances
* What's Up Docker
* Grafana
* Prometheus

Homepage is considered the **operational aggregation layer**, not the primary monitoring backend.

Historical metrics and service monitoring are provided by the dedicated observability tools.

---

# Monitoring and Observability

The HomeLab now includes a dedicated monitoring stack.

![Monitoring & Observability Architecture](../diagrams/exported/monitoring-architecture.svg)

The monitoring architecture includes:

* **Prometheus** — metrics collection and querying
* **Grafana** — dashboards and historical visualization
* **Uptime Kuma** — service availability monitoring
* **Netdata** — detailed real-time infrastructure metrics
* **Glances** — lightweight live system metrics
* **What's Up Docker** — Docker image update visibility
* **Speedtest Tracker** — WAN performance history
* **Homepage** — centralized operational overview

This layered design provides both:

* Real-time visibility
* Historical analysis
* Service-health monitoring
* Infrastructure capacity monitoring
* Update awareness

Detailed monitoring documentation is available in:

[Monitoring & Observability](monitoring.md)

---

# Backup Architecture

The lab currently uses multiple backup mechanisms.

## Proxmox Backups

Virtual machines and LXC containers are backed up to:

```text
/mnt/media/proxmox-backups
```

A snapshot-based workflow is also used before significant changes.

```text
Create Snapshot
      │
      ▼
Apply Update / Change
      │
      ▼
Test Services
   ┌──┴──┐
   │     │
Success Failure
   │     │
   ▼     ▼
Delete  Rollback
Snapshot
```

---

## Home Assistant Backups

Home Assistant backups are also copied off-site to Google Drive.

This provides an independent recovery location for one of the most important workloads.

Backup design and the previous storage recovery incident are documented in:

[Backup & Recovery](backup-recovery.md)

---

# Security Architecture

The infrastructure currently follows a low-exposure security model.

The main principles include:

* No unnecessary public port forwarding
* VPN-based remote administration
* Workload separation
* Dedicated service containers
* Authenticated SMB access
* API tokens where supported
* SMB1 disabled
* Snapshots before infrastructure changes
* Multi-layer backups
* DNS filtering through Pi-hole
* Separate testing environment

Current segmentation is mainly performed at the virtualization and service level.

Full VLAN-based network segmentation has not yet been implemented.

The planned network model includes:

```text
Management VLAN
Services VLAN
IoT VLAN
Lab VLAN
Guest VLAN
```

with explicit firewall rules controlling communication between trust zones.

Detailed security controls and the HomeLab threat model are documented in:

[Cybersecurity](cybersecurity.md)

---

# Architectural Decisions

## Why Proxmox?

Proxmox provides:

* VM and LXC support
* Web-based management
* Snapshot functionality
* Backup integration
* Linux-based administration
* Hardware passthrough
* Flexible storage management

It allows multiple infrastructure technologies to be tested on a single physical system.

---

## Why a VM for Home Assistant?

Home Assistant OS benefits from running as a complete appliance.

Using a dedicated VM preserves:

* Supervisor support
* Add-ons
* Straightforward updates
* Backup integration
* Device passthrough

---

## Why LXC for Jellyfin?

Jellyfin does not require a full virtual machine.

Using LXC provides:

* Lower memory overhead
* Lower storage overhead
* Direct Linux integration
* Straightforward GPU passthrough

---

## Why a Separate Docker LXC?

Running Docker in a dedicated LXC prevents application containers from being installed directly on the Proxmox host.

This improves:

* Organization
* Separation
* Maintainability
* Backup management

Docker inside LXC does, however, require relaxed container security settings and is therefore documented as a conscious trade-off.

---

## Why a Dedicated Pi-hole LXC?

DNS is an important infrastructure service.

Keeping Pi-hole outside the Docker host means DNS filtering can continue to operate independently of Docker maintenance or failures.

---

## Why a Separate WebApp Lab?

Experimental applications should not share the same trust level as stable infrastructure.

The dedicated WebApp LXC provides a controlled environment for:

* Development
* Deployment testing
* Hardening
* Security experiments

and can later be isolated further using VLAN segmentation.

---

## Why exFAT?

The external media disk must remain portable.

exFAT provides compatibility across:

* Linux
* Windows
* macOS

while supporting large media files.

The trade-off is that exFAT lacks several Linux-native filesystem features such as:

* Unix permissions
* Journaling
* Advanced integrity features

For a future dedicated NAS or storage server, a server-oriented filesystem would be more appropriate.

---

# Current Limitations

The current architecture is intentionally practical rather than enterprise-grade.

Known limitations include:

* Media and Proxmox backups share the same physical HDD
* No full VLAN segmentation yet
* Single physical Proxmox node
* No storage redundancy
* No high availability
* External USB storage is a single point of failure
* Docker inside LXC requires relaxed AppArmor settings
* Several services depend on the same Docker LXC
* Security monitoring is still being expanded
* No dedicated IDS/IPS or SIEM is currently deployed

These limitations are intentionally documented because they represent future learning and improvement opportunities.

---

# Future Architecture

Planned improvements include:

* Dedicated backup storage
* VLAN-based network segmentation
* Inter-VLAN firewall policies
* Dedicated Lab VLAN
* Dedicated downloader VPN isolation
* Centralized security logging
* Wazuh or Suricata integration
* Expanded Prometheus exporters
* Grafana alerting
* Automated configuration backups
* Infrastructure deployment through Ansible
* Secrets management
* Internal HTTPS expansion
* Improved backup testing
* Additional controlled cybersecurity workloads

The long-term objective is to evolve the HomeLab from a single-node self-hosted environment into a more segmented, observable, secure, and reproducible infrastructure platform while maintaining reasonable complexity and operating cost.
