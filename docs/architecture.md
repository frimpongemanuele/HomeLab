# HomeLab Architecture

## Overview

This document describes the architecture of my self-hosted HomeLab environment.

The platform is built around **Proxmox VE** running on a Lenovo ThinkCentre M720q Tiny and is designed to provide a modular environment for virtualization, smart home automation, media services, secure remote access, monitoring, and cybersecurity experimentation.

The infrastructure evolved from an older laptop-based server into a dedicated mini-PC platform to improve reliability, isolation, scalability, and maintainability.

The current design follows several core principles:

* Separate major workloads using virtual machines and LXC containers
* Keep management services private
* Avoid unnecessary public exposure
* Use centralized shared storage for media and backups
* Maintain portability of the external storage device
* Keep the environment simple enough to maintain while still allowing future expansion

---

## High-Level Architecture

The Proxmox host is the central compute node of the environment.

```text
Internet
   │
   ├── Tailscale VPN
   └── WireGuard VPN
           │
           ▼
      Home Network
           │
      TIM HUB+ Router
           │
           ▼
    Proxmox VE Host
           │
    ┌──────┼───────────────┐
    │      │               │
    ▼      ▼               ▼
HAOS VM  Jellyfin LXC   Docker LXC
                           │
              ┌────────────┼────────────┐
              │            │            │
            Radarr       Sonarr      Prowlarr
              │            │            │
              └────────────┼────────────┘
                           │
                       qBittorrent
                           │
                           ▼
                    Shared Storage
```

The system is intentionally divided into multiple logical layers.

---

## Physical Layer

### Proxmox Host

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

The system was selected because of its relatively low power consumption, compact size, Intel virtualization support, and integrated Intel GPU.

The UHD Graphics 630 is also used by Jellyfin for hardware-accelerated video transcoding through VAAPI.

---

## Network Layer

The HomeLab currently operates inside the main home LAN.

The primary router is a **TIM HUB+**, which provides:

* Internet gateway
* DHCP
* Local DNS
* NAT
* Wi-Fi connectivity
* Gigabit Ethernet connectivity

The Proxmox host is assigned a stable internal address using a static configuration or DHCP reservation.

Internal services are not intentionally exposed through public port forwarding.

Remote access is instead provided through encrypted VPN solutions.

### Remote Access

Two VPN technologies are currently used within the environment:

* **Tailscale**
* **WireGuard**

Tailscale is used as the main secure remote-access mechanism for infrastructure services.

WireGuard is also available through Home Assistant for remote connectivity.

This allows access to services such as:

* Proxmox
* Home Assistant
* Jellyfin
* Homepage
* Internal management interfaces

without exposing their administration ports directly to the public Internet.

---

## Virtualization Layer

Proxmox VE provides the virtualization layer for the lab.

Different workloads are separated according to their requirements.

### Virtual Machine

| ID  | Workload          | Type |
| --- | ----------------- | ---- |
| 100 | Home Assistant OS | VM   |

### LXC Containers

| ID  | Workload  | Role                           |
| --- | --------- | ------------------------------ |
| 101 | Jellyfin  | Media server                   |
| 102 | Docker    | Containerized application host |
| 103 | Tailscale | Remote-access service          |
| 104 | Homepage  | Centralized dashboard          |

Additional monitoring and security workloads may be deployed as the environment evolves.

---

## Home Assistant VM

Home Assistant OS runs as a dedicated virtual machine rather than inside a container.

This provides a complete Home Assistant appliance environment and simplifies:

* Add-on management
* Integrations
* USB device access
* Backup and restore operations
* Smart home device management

The VM also interfaces with Zigbee devices through a **SONOFF Zigbee 3.0 USB Dongle Plus**.

A Philips Hue Bridge remains part of the smart lighting environment.

---

## Jellyfin LXC

Jellyfin runs inside a dedicated LXC container.

Using LXC provides lower overhead than a full virtual machine while still separating the media server from the Proxmox host.

The Jellyfin container receives access to:

```text
/dev/dri/card0
/dev/dri/renderD128
```

This enables VAAPI-based hardware transcoding using the Intel UHD Graphics 630 integrated GPU.

The external media storage is exposed to the container through a Proxmox bind mount.

From the Jellyfin container, the media libraries are available under:

```text
/media/movies
/media/tv
```

Further Jellyfin configuration is documented in:

[`jellyfin.md`](jellyfin.md)

---

## Docker Application Layer

A dedicated LXC container is used as a Docker host.

This isolates Docker workloads from both the Proxmox host and the Jellyfin server.

The container currently hosts the media automation stack through Docker Compose.

The main services include:

| Service     | Purpose              |
| ----------- | -------------------- |
| Prowlarr    | Indexer management   |
| Radarr      | Movie management     |
| Sonarr      | TV series management |
| Bazarr      | Subtitle management  |
| qBittorrent | Download client      |

The logical media workflow is:

```text
Prowlarr
   │
   ├── Radarr
   └── Sonarr
         │
         ▼
     qBittorrent
         │
         ▼
      Downloads
         │
         ▼
   Media Library
         │
         ▼
      Jellyfin
```

The Docker implementation is documented separately in:

[`docker-media-stack.md`](docker-media-stack.md)

---

## Shared Storage Architecture

The lab uses an external **3 TB Seagate ST3000DM001 HDD** connected through an ICY BOX USB enclosure.

The drive is mounted on the Proxmox host at:

```text
/mnt/media
```

The current filesystem is **exFAT**.

exFAT was intentionally selected because the drive must remain portable and readable from:

* Linux / Proxmox
* Windows
* macOS
* Other computers if required

The current storage structure is approximately:

```text
/mnt/media/
├── movies/
├── tv/
└── proxmox-backups/
```

The same physical HDD currently stores both media and Proxmox backups.

This is convenient and functional for the current lab, but it creates a shared failure domain: failure of the disk would affect both the media library and the local infrastructure backups.

For this reason, separating media storage and backup storage is planned as a future improvement.

---

## Storage Mount Strategy

Because the storage device is external, the Proxmox host must remain bootable if the disk is disconnected or unavailable.

The drive is therefore mounted using UUID-based configuration and the `nofail` option.

Example:

```fstab
UUID=XXXX-XXXX /mnt/media exfat defaults,nofail,uid=1000,gid=1000,umask=000 0 0
```

The `nofail` option was introduced after a previous mount configuration caused Proxmox to enter emergency mode when the external drive could not be mounted during startup.

This experience directly influenced the final storage design.

---

## File Sharing

The media storage is also exposed to Windows systems using Samba.

Example logical path:

```text
\\PROXMOX-HOST\media
```

A dedicated Samba account is used instead of anonymous guest access.

SMB1 is disabled because it is obsolete and insecure.

The Samba layer allows media management and file transfers from Windows without requiring direct physical access to the external HDD.

---

## Dashboard and Observability

Homepage is deployed as a centralized dashboard for accessing and monitoring HomeLab services.

The dashboard integrates with services such as:

* Proxmox
* Home Assistant
* Jellyfin
* Other internal applications

Monitoring has also been expanded with tools including:

* Prometheus
* Grafana
* Uptime Kuma
* Proxmox metrics
* Jellyfin metrics
* Home Assistant metrics

These services provide greater visibility into availability, performance, and resource usage.

---

## Backup Architecture

The lab currently uses multiple backup mechanisms.

### Proxmox

Virtual machines and containers are backed up to:

```text
/mnt/media/proxmox-backups
```

A snapshot-based workflow is also used before significant changes:

```text
Create snapshot
      │
      ▼
Apply update/change
      │
      ▼
Test services
   ┌──┴──┐
   │     │
   OK   Failure
   │     │
   ▼     ▼
Delete Rollback
snapshot
```

### Home Assistant

Home Assistant backups are also copied off-site using Google Drive.

This provides a second recovery location for one of the most important workloads.

Backup design and the storage recovery incident are documented in:

[`backup-recovery.md`](backup-recovery.md)

---

## Security Architecture

The infrastructure currently follows a low-exposure model.

The main principles are:

* No unnecessary public port forwarding
* VPN-based remote administration
* Separation of workloads
* Dedicated API tokens where possible
* Authenticated SMB access
* Legacy SMB1 disabled
* Snapshots before infrastructure changes
* Backup of critical workloads

Current segmentation is primarily performed at the virtualization and service level.

Full network segmentation using dedicated VLANs is planned but has not yet been implemented.

The planned model includes:

```text
Management VLAN
IoT VLAN
Media VLAN
Guest VLAN
```

with explicit firewall rules controlling communication between zones.

Detailed security controls and the lab threat model are documented in:

[`cybersecurity.md`](cybersecurity.md)

---

## Design Decisions

### Why Proxmox?

Proxmox provides:

* VM and LXC support
* Web-based management
* Snapshot functionality
* Backup integration
* Linux-based administration
* Flexible hardware passthrough

It allows multiple infrastructure technologies to be tested on a single physical machine.

### Why LXC for Jellyfin?

Jellyfin does not require a complete virtual machine.

Using LXC provides:

* Lower memory overhead
* Lower storage overhead
* Direct Linux integration
* Straightforward GPU device passthrough

### Why a Separate Docker LXC?

Running Docker separately avoids mixing application containers with the Proxmox host.

This improves organization and reduces the number of applications installed directly on the hypervisor.

### Why exFAT?

The external media disk needs to remain portable.

Using exFAT provides native or straightforward compatibility across multiple operating systems while supporting large media files.

The trade-off is that exFAT lacks many Linux-native filesystem features such as Unix permissions, journaling, and advanced integrity mechanisms.

For a future dedicated storage server or NAS, a filesystem better suited to server workloads would be preferable.

---

## Current Limitations

The present architecture is intentionally practical rather than enterprise-grade.

Current limitations include:

* Media and local Proxmox backups share the same physical HDD
* No full VLAN segmentation yet
* Single physical Proxmox node
* No storage redundancy
* No high availability
* Docker inside LXC requires relaxed AppArmor configuration
* External USB storage represents a single point of failure

These limitations are documented rather than hidden because they represent areas for future experimentation and improvement.

---

## Future Architecture

Planned improvements include:

* Dedicated backup storage
* VLAN-based network segmentation
* Inter-VLAN firewall policies
* Gluetun VPN isolation for download traffic
* Centralized security logging
* Wazuh or Suricata integration
* Automated configuration backups
* Infrastructure deployment using Ansible
* Secret management
* Expanded monitoring and alerting

The long-term objective is to evolve the HomeLab from a single-node self-hosted environment into a more segmented and observable infrastructure platform while preserving simplicity and low operational cost.
