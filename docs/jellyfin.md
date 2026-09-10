# Jellyfin Media Server

## Overview

Jellyfin is the primary media-streaming platform of the HomeLab.

It provides a self-hosted interface for organizing and streaming movies, television series, and other media stored on the external media disk.

Unlike the media automation applications, Jellyfin does **not** run inside the Docker LXC.

Instead, it runs in a dedicated LXC container on Proxmox VE.

This separation provides:

- Independent resource allocation
- Independent updates and restarts
- Direct access to the shared media storage
- GPU access for hardware-accelerated transcoding
- Separation from download and automation workloads
- Easier troubleshooting
- Reduced dependency on the Docker environment

---

# Architecture

The Jellyfin architecture separates media acquisition from media consumption.

```text
                   Media Automation
                        Docker
                           │
               Sonarr / Radarr / Bazarr
                           │
                           ▼
                    Shared Storage
                     /mnt/media
                           │
               ┌───────────┴───────────┐
               │                       │
          /media/movies            /media/tv
               │                       │
               └───────────┬───────────┘
                           │
                           ▼
                    Jellyfin LXC
                           │
                           ▼
                     Home Network
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
               TV        Browser    Mobile
```

Jellyfin is therefore the final presentation layer of the media pipeline.

The complete acquisition workflow is documented in:

[Docker & Media Stack](docker-media-stack.md)

---

# Proxmox Deployment

Jellyfin runs inside a dedicated LXC container.

| Property | Configuration |
|---|---|
| Hypervisor | Proxmox VE |
| Container ID | `101` |
| Service | Jellyfin |
| Deployment | Dedicated LXC |
| Network | Home LAN |
| Media Storage | External 3 TB HDD |
| Media Mount | `/media` |
| Web Interface | Port `8096` |
| Hardware Acceleration | Intel GPU / VA-API |

The separation can be represented as:

```text
             Proxmox VE
                 │
       ┌─────────┴─────────┐
       │                   │
    LXC 101             LXC 102
    Jellyfin             Docker
       │                   │
       │              Media Automation
       │                   │
       └─────────┬─────────┘
                 │
                 ▼
          Shared Media Storage
```

This allows Jellyfin to remain available independently of the Docker media-management stack.

---

# Why a Dedicated LXC?

Several deployment models were considered.

Jellyfin could have been deployed:

- Directly on the Proxmox host
- Inside Docker
- Inside a virtual machine
- Inside a dedicated LXC

A dedicated LXC provides a useful balance between isolation and resource efficiency.

## Advantages

### Lightweight

LXC containers have lower overhead than full virtual machines.

### Workload Separation

Jellyfin does not share its application environment with Sonarr, Radarr, qBittorrent, or other Docker services.

### Independent Lifecycle

Jellyfin can be:

- Updated
- Restarted
- Backed up
- Snapshotted
- Troubleshot

without restarting the Docker stack.

### Hardware Access

Required GPU devices can be passed into the container for hardware-accelerated transcoding.

---

# Storage Architecture

The media library resides on the external 3 TB HDD connected to the Proxmox host.

At the host level, the storage is mounted at:

```text
/mnt/media
```

The disk contains media directories such as:

```text
/mnt/media/
├── movies/
├── tv/
└── proxmox-backups/
```

The relevant media directories are made available to the Jellyfin LXC through a bind mount.

Conceptually:

```text
External HDD
     │
     ▼
Proxmox Host
/mnt/media
     │
     │ Bind Mount
     ▼
Jellyfin LXC
/media
     │
     ├── /media/movies
     └── /media/tv
```

This allows the storage to remain managed by the Proxmox host while Jellyfin receives access only to the required path.

---

# LXC Bind Mount

A Proxmox bind mount can expose host storage inside an LXC container.

Conceptually:

```text
Host:
    /mnt/media

        │
        ▼

LXC 101:
    /media
```

A corresponding Proxmox container configuration can resemble:

```text
mp0: /mnt/media,mp=/media
```

This avoids duplicating media inside the Jellyfin container's root filesystem.

It also keeps application data and media data logically separate.

---

# Media Library Structure

A predictable directory structure is important for reliable metadata matching.

## Movies

```text
/media/movies/
└── Movie Name (Year)/
    └── Movie Name (Year).mkv
```

Example:

```text
/media/movies/
└── Example Movie (2026)/
    └── Example Movie (2026).mkv
```

## Television

```text
/media/tv/
└── Series Name/
    ├── Season 01/
    │   ├── Series Name S01E01.mkv
    │   └── Series Name S01E02.mkv
    │
    └── Season 02/
        └── Series Name S02E01.mkv
```

This structure is also compatible with the media automation workflow used by Sonarr and Radarr.

---

# Media Automation Integration

Jellyfin is deliberately separated from media acquisition.

The media automation stack handles:

```text
Requests
   ↓
Discovery
   ↓
Downloading
   ↓
Importing
   ↓
Renaming
   ↓
Library organization
```

Jellyfin handles:

```text
Library scanning
   ↓
Metadata
   ↓
Playback
   ↓
Transcoding when required
```

The complete flow is:

```text
Jellyseerr
     │
     ▼
Sonarr / Radarr
     │
     ▼
Prowlarr
     │
     ▼
qBittorrent
     │
     ▼
Downloads
     │
     ▼
Import / Rename
     │
     ▼
/media/movies
/media/tv
     │
     ▼
Jellyfin
     │
     ▼
Client Device
```

This separation makes each component easier to maintain and troubleshoot.

---

# Hardware-Accelerated Transcoding

Media clients do not always support the original video codec, resolution, bitrate, or audio format.

When direct playback is not possible, Jellyfin may need to transcode the media.

Software transcoding uses the CPU:

```text
Media File
    │
    ▼
CPU Transcoding
    │
    ▼
Compatible Stream
```

This can consume significant CPU resources.

The HomeLab therefore uses the integrated Intel graphics available on the Proxmox host for hardware acceleration.

---

# Intel GPU Passthrough

The host exposes the Intel Direct Rendering Infrastructure devices under:

```bash
/dev/dri/
```

A typical system may expose:

```text
/dev/dri/card0
/dev/dri/renderD128
```

The render device is passed through to the Jellyfin LXC.

Conceptually:

```text
Intel Integrated GPU
        │
        ▼
    Proxmox Host
        │
   /dev/dri/renderD128
        │
        ▼
    Jellyfin LXC
        │
        ▼
       VA-API
        │
        ▼
Hardware Transcoding
```

This allows video processing to be offloaded from the CPU.

---

# VA-API

Jellyfin uses VA-API to access the Intel graphics hardware.

The configured environment supports hardware acceleration for common codecs such as:

- H.264
- HEVC
- VP9

Hardware acceleration reduces CPU utilization during compatible transcoding workloads.

The architecture becomes:

```text
Original Media
      │
      ▼
   Jellyfin
      │
      ▼
    VA-API
      │
      ▼
 Intel iGPU
      │
      ▼
Encoded Stream
      │
      ▼
Client Device
```

---

# Direct Play vs Transcoding

Jellyfin prefers **Direct Play** whenever the client supports the original media.

```text
Media
  │
  ├── Compatible ──────► Direct Play
  │
  └── Incompatible ────► Transcoding
                              │
                              ▼
                          Intel iGPU
```

Direct Play is preferable because it:

- Uses fewer server resources
- Avoids quality loss
- Reduces transcoding overhead
- Simplifies playback

Hardware transcoding provides a fallback for clients that cannot play the original media directly.

---

# Permissions

Storage permissions are an important part of the Jellyfin deployment.

Three layers interact with the media disk:

```text
Proxmox Host
      │
      ├── Jellyfin LXC
      │
      ├── Docker Media Stack
      │
      └── Samba
```

All required services must be able to access the same media files without creating incompatible ownership or permission states.

Because the external media disk uses exFAT, traditional Unix ownership and permission behavior is more limited than with Linux-native filesystems such as ext4 or ZFS.

Access is therefore managed primarily through:

- Mount options
- LXC bind mounts
- Samba authentication
- Application-level permissions

This trade-off was accepted because portability between Linux and Windows was an important design requirement.

---

# Samba and Jellyfin

The Samba share is provided by the Proxmox host rather than by the Jellyfin container.

This distinction is intentional.

```text
                     External HDD
                          │
                          ▼
                     Proxmox Host
                      /mnt/media
                      /         \
                     /           \
                    ▼             ▼
             Jellyfin LXC       Samba
                    │             │
                    ▼             ▼
               Media Server   Windows Client
```

Jellyfin therefore does not need to act as a general-purpose file server.

This follows the principle of separating application responsibilities.

---

# Networking

Jellyfin is available on the local network through its dedicated LXC address.

The default Jellyfin HTTP service uses:

```text
TCP 8096
```

Conceptually:

```text
Client
   │
   ▼
Home LAN
   │
   ▼
Jellyfin LXC
   │
   ▼
TCP 8096
```

The management interface is not intentionally exposed directly to the public Internet.

---

# Remote Access

Remote access to the HomeLab is handled through VPN infrastructure rather than direct Jellyfin port forwarding.

```text
Remote Client
      │
      ▼
Tailscale / WireGuard
      │
      ▼
   Home LAN
      │
      ▼
 Jellyfin
```

This reduces the need to expose Jellyfin directly at the network perimeter.

It also keeps remote-access policy separate from the media application itself.

---

# Security Considerations

Jellyfin processes complex media files and exposes a web application, making it part of the HomeLab attack surface.

Potential risks include:

- Web application vulnerabilities
- Weak user credentials
- Plugin vulnerabilities
- Malicious media files
- Media-parser vulnerabilities
- Excessive filesystem permissions
- Accidental Internet exposure

Current controls include:

- Dedicated LXC
- Private LAN placement
- VPN-based remote access
- Separation from Docker workloads
- Limited GPU-device passthrough
- Media storage separated from application storage
- No unnecessary direct public administration

More details are documented in:

[Cybersecurity Design](cybersecurity.md)

---

# Updates

Jellyfin is maintained independently from the Docker stack.

A typical update workflow is:

```text
Check current version
       │
       ▼
Create backup / snapshot
       │
       ▼
Update package repository
       │
       ▼
Upgrade Jellyfin
       │
       ▼
Restart service
       │
       ▼
Verify playback
       │
       ▼
Verify hardware acceleration
```

Example package-management commands on a Debian/Ubuntu-based environment include:

```bash
apt update
apt upgrade
```

After an update, the service can be verified with:

```bash
systemctl status jellyfin
```

and the installed version with:

```bash
jellyfin --version
```

Major upgrades should be preceded by a recoverable backup or snapshot.

---

# Backup Strategy

Jellyfin recovery involves two different categories of data.

## Application Configuration

Important Jellyfin application data includes:

```text
Configuration
Database
Metadata
Plugins
User accounts
Playback state
```

These should be protected through Proxmox LXC backups and, where appropriate, application-level configuration backups.

## Media

The media library resides outside the LXC on the external HDD.

This means:

```text
LXC Backup
    ≠
Media Backup
```

Backing up the Jellyfin container protects the application environment but does not automatically create another copy of the media library.

This distinction is important when designing recovery procedures.

More details are documented in:

[Backup & Recovery](backup-recovery.md)

---

# Monitoring

Jellyfin can be monitored at multiple layers.

```text
Proxmox
   │
   └── LXC resource usage
           │
           ▼
       Jellyfin
           │
           ├── Availability
           ├── Playback
           └── Application logs
```

The wider HomeLab monitoring stack provides visibility into:

- Container availability
- CPU utilization
- Memory utilization
- Storage utilization
- Network activity
- Service uptime

Uptime Kuma can provide service-level availability monitoring, while Proxmox, Prometheus, Grafana, Netdata, and Glances provide infrastructure-level visibility.

---

# Troubleshooting Methodology

Jellyfin troubleshooting is approached from the infrastructure layer upward.

```text
Proxmox Host
     │
     ▼
LXC Running?
     │
     ▼
Network Reachable?
     │
     ▼
Storage Mounted?
     │
     ▼
Jellyfin Service Running?
     │
     ▼
Library Accessible?
     │
     ▼
GPU Available?
     │
     ▼
Client Playback
```

This avoids assuming that every playback problem originates inside Jellyfin.

Useful checks include:

```bash
systemctl status jellyfin
```

```bash
journalctl -u jellyfin
```

```bash
ls -la /media
```

```bash
ls -la /dev/dri
```

```bash
vainfo
```

These checks help distinguish between:

- Application failures
- Storage problems
- Permission problems
- GPU passthrough problems
- Network issues
- Client compatibility problems

---

# Storage Incident and Recovery

The media storage used by Jellyfin was involved in a significant filesystem recovery incident during the evolution of the HomeLab.

Filesystem metadata was damaged while reconfiguring the external disk.

The recovery process required:

- TestDisk investigation
- R-Studio analysis
- PhotoRec file carving
- Media-file verification
- Manual library reconstruction
- Metadata-based identification
- Rebuilding Jellyfin-compatible folder structures

This experience directly influenced the current storage procedures.

Important lessons included:

```text
Inspect disks before modifying them
Mount unknown filesystems read-only first
Avoid destructive commands until the target is verified
Keep recovery source data untouched
Separate backups from primary storage
```

The complete incident is documented in:

[Backup & Recovery](backup-recovery.md)

---

# Design Principles

## Separate Media Serving from Media Acquisition

Jellyfin serves content.

Sonarr, Radarr, Prowlarr, qBittorrent, and related applications acquire and organize content.

These responsibilities remain separate.

## Keep Media Outside the Application Container

The media library is mounted into Jellyfin rather than stored inside the LXC root filesystem.

## Prefer Direct Play

Compatible clients should play the original media whenever possible.

## Use Hardware Acceleration When Transcoding Is Required

Intel VA-API reduces CPU load for supported codecs.

## Keep Remote Access Private

VPN-based remote access is preferred over unnecessary public exposure.

## Maintain Recoverability

Application configuration and media storage have different backup requirements and should be treated separately.

---

# Future Improvements

Potential improvements include:

- Dedicated NAS or storage server
- Separate backup storage
- Linux-native filesystem for stronger permission management
- Improved storage redundancy
- Automated media-integrity checks
- Additional monitoring around transcoding
- GPU utilization dashboards
- More granular Jellyfin metrics
- Improved backup validation
- Internal HTTPS
- Network segmentation
- Dedicated Services VLAN
- Restrict Jellyfin access to required network zones
- Improved client compatibility and Direct Play coverage

---

# Skills Demonstrated

The Jellyfin deployment demonstrates practical experience with:

- Linux containers
- Proxmox LXC administration
- Bind mounts
- Linux storage
- Shared filesystems
- Samba
- Media-server administration
- Intel GPU passthrough
- VA-API
- Hardware-accelerated transcoding
- Linux permissions
- Network services
- VPN-based remote access
- Backup planning
- Service troubleshooting
- Filesystem recovery

---

# Key Takeaways

Jellyfin is deliberately treated as an independent infrastructure workload rather than simply another Docker application.

The architecture combines:

```text
Dedicated LXC
      +
Shared Storage
      +
Intel GPU
      +
VA-API
      +
Media Automation
      +
Private Networking
      +
Monitoring
      +
Backup & Recovery
```

Separating media serving from acquisition and storage management makes the system easier to operate, troubleshoot, secure, and evolve.

---

# Related Documentation

- [Architecture](architecture.md)
- [Docker & Media Stack](docker-media-stack.md)
- [Networking](networking.md)
- [Cybersecurity](cybersecurity.md)
- [Monitoring](monitoring.md)
- [Backup & Recovery](backup-recovery.md)
