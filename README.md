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
- Proxmox Installation
Proxmox VE was installed on a Lenovo mini PC used as the main virtualization layer.

Main storage:
NVMe SSD → Proxmox system storage
External 3TB HDD → media and backup storage

### Home Assistant VM
Home Assistant OS was deployed as a VM.

Key points:
•	Restored from an existing backup
•	Used for smart home automation
•	WireGuard configured for remote access
•	Google Drive backup configured for off-site backup

### Jellyfin LXC
Jellyfin was installed in an LXC container.

Container details:
Container ID: 101
Hostname: jellyfin
IP address: 192.168.XXX.XXX
Port: 8096
Jellyfin libraries:
Movies → /movies
TV     → /tv

- Intel GPU Passthrough for Jellyfin
The Proxmox host exposed Intel GPU devices:
/dev/dri/card0
/dev/dri/renderD128

The Jellyfin user was added to the required groups:
usermod -aG video jellyfin
usermod -aG render jellyfin
VAAPI was enabled in Jellyfin:
Hardware acceleration: VAAPI
VAAPI device: /dev/dri/renderD128
Hardware encoding: Enabled
Tone mapping: Disabled initially

Recommended decoding options:
Enabled:
- H264
- HEVC
- HEVC 10-bit
- VP9
- VP9 10-bit

Disabled initially:
- AV1
- Low-power H264 encoder
- Low-power HEVC encoder
- Tone mapping

### Docker LXC
Docker was installed inside LXC 102.

To support nested Docker, the LXC configuration was modified:
features: nesting=1,keyctl=1
lxc.apparmor.profile: unconfined
For specific Docker containers, AppArmor was disabled:
security_opt:
  - apparmor=unconfined
This was required because Docker inside LXC can conflict with AppArmor profiles.

## Media Automation Stack

The following services were deployed with Docker Compose:
qBittorrent
Radarr
Sonarr
Prowlarr
Bazarr

Media pipeline:
Prowlarr → Sonarr/Radarr → qBittorrent → Downloads → Media Library → Jellyfin

Path consistency was critical:
/media/movies
/media/tv

All containers were configured to use consistent media paths to avoid import errors.

## External HDD Mount
The final storage design uses exFAT for portability.

Example mount workflow:
apt update
apt install -y exfatprogs

mkdir -p /mnt/media
lsblk -f
blkid

Safe /etc/fstab entry:
UUID=XXXX-XXXX /mnt/media exfat defaults,nofail,uid=1000,gid=1000,umask=000 0 0
The nofail option is important because it prevents Proxmox from entering emergency mode if the external disk is disconnected during boot.

## Bind Mount to Jellyfin

The media drive is bind-mounted into the Jellyfin container:
pct set 101 -protection 0
pct set 101 -mp0 /mnt/media,mp=/media
pct set 101 -protection 1

Inside the container:
ls /media

Expected result:
movies
tv

# Cybersecurity Focus
Security was one of the main design goals of this home lab.

Implemented Security Measures
Remote Access Without Port Forwarding
No services are directly exposed to the public internet.

Remote access is handled through:
•	Tailscale
•	WireGuard

This reduces the attack surface significantly.
Public Internet
    │
    └── No exposed Jellyfin / Proxmox / Home Assistant ports


