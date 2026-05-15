# HomeLab
Secure Home Lab Infrastructure with Proxmox (Home Assistant, Media Server &amp; VPN)

This repository is intended to document my journey in setting up a HomeLab, with the goal of hosting services and tools.
It will also be the fist project I'll document fully here on Github.

It will be touching all service that can come to mind in a standard home server istance and it will document the design and implementation of a self-hosted home lab build around **Proxmox VE**, focused on virtualization, cybersecurity, networking, automation, media services, secure remote access and backup strategy.

My fist lab was setup simply with an older laptop based server which is being replaced by a scalable mini PC infrastructure and an external storage device.

<p> With time, the goal is to experiment and learn about securing it and improving networking and storage. Main goals: </p>

- Build a reliable self-hosted infrastracture;
- Practice virtualization and containerization;
- Deployment of smart home and media services;
- Implementation of secure remote access without exposing services publicly;
- Centralized monitoring through a dashboard;
- Application of cybersecurity best practices;
- Creation of a realistic enviroment for learning DevOps, networking and system administration.

## Hardware:

- Lenovo ThinkCentre M710q Tiny (Main Proxmox host)
  - Intel Core i5-8500T, 6 Cores, 6 Threads, 2.1 GHz, Up to 3.5 GHz, 35W (Coffee Lake)
  - Intel UHD Graphics 630 (Integrated)
  - Proprietary Tiny motherboard (Q370 chipset)
  - 16 GB DDR4 SO-DIMM
  - 256 GB NVMe SSD

- External 3TB HDD, Seagate ST3000DM001 (3TB 3.5" SATA HDD)

- ICY BOX IB-377U3 (External 3.5" HDD SATA enclosure), USB 3.0, SATA III up to 6 Gbit/s, UASP support, external 12V power supply

- TIM HUB+ Router (Main home router), Wi-Fi 6, dual-band 2.4 GHz/5 GHz, Gigabit Ethernet, EasyMesh support

- SONOFF Zigbee 3.0 USB Dongle Plus (20 dBm output gain)

- Philips Hue Bridge 2.0

## Key Technologies:

- Proxmox VE
- Home Assistant OS VM
- Jellyfin LXC
- Docker LXC
- Tailscale LXC
- Homepage Dashboard
- Monitoring and Security workloads

## Network Topology:

- [Internet]
  - No public port forwarding
- [Tailscale VPN / WireGuard]
  - [Home LAN]
- [Proxmox Host]
  - Home Assistant VM
  - Jellyfin LXC
  - Docker LXC
  - Tailscale LXC
  - Homepage Dashboard
  - Security

## Architecture Explanation
The Proxmox host acts as the core infrastracture layer. Each major service is isolated either in a VM or LXC container.

Home Assistant runs as a dedicated VM because HA OS benefits from a full appliance-style environment and add-on support.

Jellyfin runs inside an LXC container for lightweight performance adn direct access to media storage. Intel GPU passthrough is configured for hardware-accelerated transcoding.

Docker is deployed inside a separate LXC container. This container hosts the media automation stack, keeping Docker workloads isolated from both Proxmox host and Jellyfin.
Remote access is handled through Tailscale and WireGuard, avoiding unsage public exposure of internal services.

## Tech Stack

| Technology | Purpose | Reason |
| ------------- | ------------- | ------------- |
| Proxmox VE | Hypervisor | Reliable virtualization platform with VM, LXC, backup and snapshot support |
| Home Assistant OS | Smart home platform | Centralized automation and IoT control |
| LXC | Lightwight containers | Efficent isolation for Linux services |
| Docker | Application deployment | Easy service deployment with Docker Compose |
| Jellyfin | Media Server | Self-hosted streaming platform |

<br>

| Technology | Purpose | Reason |
| ------------- | ------------- | ------------- |
| Radarr | Movie automation	| Manages movie library |
| Sonarr	| TV automation	| Manages TV show library |
| Prowlarr	| Indexer manager	| Centralized indexer configuration |
| Bazarr	| Subtitle automation	| Automatic subtitle management |
| Tailscale	| Secure remote access	| Zero-trust VPN without port forwarding |
| WireGuard	| VPN access	| Secure remote access to the home network |
| Samba	| File sharing	| Allows Windows clients to access media storage |
| Homepage	| Dashboard	| Centralized monitoring and service access |
| exFAT	| Portable HDD filesystem	| Compatible with Linux, Windows and macOS |

## Setup & Deployment



