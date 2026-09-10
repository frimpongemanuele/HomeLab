# Docker & Media Stack

## Overview

The Docker environment hosts most of the application-level services in my HomeLab.

Docker runs inside a dedicated **Proxmox LXC container (LXC 102)**, allowing application workloads to remain separated from:

- The Proxmox hypervisor
- Home Assistant
- Jellyfin
- Pi-hole
- Tailscale
- Experimental WebApp workloads

The Docker host is used for:

- Media automation
- Download management
- Media requests
- IPTV / live TV tooling
- Reverse proxying
- Monitoring
- WAN performance tracking
- Container update visibility

Docker Compose is used to define and manage the services in a reproducible way.

---

# Architecture

The Docker layer sits inside the broader Proxmox virtualization environment:

```text
Physical Host
    │
    ▼
Proxmox VE
    │
    ▼
LXC 102 - Docker
    │
    ▼
Docker Engine
    │
    ├── Media Services
    ├── Networking Services
    ├── Monitoring Services
    └── Maintenance Services
```

This design keeps most application workloads off the Proxmox host while still avoiding the overhead of creating a separate virtual machine for every service.

---

# Why Docker Inside LXC?

Docker could have been deployed in a dedicated VM.

Instead, I chose to run Docker inside an LXC container to reduce resource overhead while still keeping application workloads separated from the Proxmox host.

Benefits include:

- Lower RAM overhead than a full VM
- Lower storage overhead
- Fast container startup
- Easy backup of the Docker LXC
- Centralized application management
- Reusable Docker Compose configurations

The architecture is:

```text
Proxmox
   │
   ▼
LXC Container
   │
   ▼
Docker Engine
   │
   ▼
Application Containers
```

This introduces an additional layer of nesting, which required specific LXC configuration.

---

# LXC Configuration for Docker

Docker initially encountered AppArmor-related restrictions inside the LXC container.

The Docker LXC configuration was adjusted to support nested containerization.

Example:

```text
features: nesting=1,keyctl=1
lxc.apparmor.profile: unconfined
```

Some Docker containers also required:

```yaml
security_opt:
  - apparmor=unconfined
```

These changes allowed Docker to operate correctly inside the LXC environment.

---

## Security Trade-Off

Relaxing AppArmor restrictions improves compatibility, but reduces one layer of security confinement.

This is therefore treated as a deliberate architectural trade-off rather than a neutral configuration change.

The current model provides:

```text
Physical Host
    │
Proxmox Kernel
    │
LXC Boundary
    │
Docker Boundary
    │
Application
```

However, LXC containers share the Proxmox host kernel.

For workloads requiring stronger isolation, a dedicated virtual machine would provide a stronger security boundary.

A possible future improvement is therefore to migrate higher-risk Docker workloads to a dedicated VM.

---

# Service Inventory

The Docker environment currently contains services across several functional groups.

## Media Automation

| Service | Role |
|---|---|
| Sonarr | TV series management |
| Radarr | Movie management |
| Prowlarr | Indexer management |
| Bazarr | Subtitle automation |
| qBittorrent | Download client |
| Jellyseerr | Media request management |
| Dispatcharr | IPTV and live TV management |

## Networking

| Service | Role |
|---|---|
| Nginx Proxy Manager | Reverse proxy and internal service routing |
| Speedtest Tracker | WAN performance monitoring |

## Monitoring

| Service | Role |
|---|---|
| Prometheus | Metrics collection |
| Grafana | Metrics dashboards and visualization |
| Uptime Kuma | Availability monitoring |
| Netdata | Detailed real-time system metrics |
| Glances | Lightweight live metrics |

## Maintenance

| Service | Role |
|---|---|
| What's Up Docker | Docker image update visibility |

---

# Media Automation Architecture

The media automation stack separates different responsibilities across several applications.

```text
                   Jellyseerr
                       │
                       ▼
             ┌─────────┴─────────┐
             │                   │
           Sonarr              Radarr
          TV Shows             Movies
             │                   │
             └─────────┬─────────┘
                       │
                       ▼
                    Prowlarr
                 Indexer Manager
                       │
                       ▼
                  qBittorrent
                 Download Client
                       │
                       ▼
                    Downloads
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
           Sonarr              Radarr
          Import TV          Import Movies
             │                   │
             ▼                   ▼
         /media/tv         /media/movies
             │                   │
             └─────────┬─────────┘
                       ▼
                    Jellyfin
```

Bazarr operates alongside the media library to manage subtitle acquisition and organization.

Dispatcharr provides IPTV and live TV management independently from the normal Sonarr/Radarr workflow.

---

# Jellyseerr

Jellyseerr provides a user-friendly request layer for the media environment.

Instead of manually adding titles directly inside Sonarr or Radarr, users can request content through Jellyseerr.

The request flow is:

```text
User
 │
 ▼
Jellyseerr
 │
 ├── TV Request ─────► Sonarr
 │
 └── Movie Request ──► Radarr
```

This separates user interaction from backend automation.

---

# Sonarr

Sonarr manages the TV series library.

Its responsibilities include:

- Monitoring TV series
- Tracking missing episodes
- Managing quality profiles
- Sending searches through configured indexers
- Sending downloads to qBittorrent
- Importing completed episodes
- Renaming and organizing files

The final TV library is stored under:

```text
/media/tv
```

A typical structure is:

```text
/media/tv/
└── Show Name/
    └── Season 01/
        └── Show Name S01E01.mkv
```

This structure is compatible with Jellyfin's TV library scanner.

---

# Radarr

Radarr performs the same type of automation for movies.

Its responsibilities include:

- Monitoring movies
- Tracking missing titles
- Managing quality profiles
- Searching through configured indexers
- Sending downloads to qBittorrent
- Importing completed downloads
- Renaming and organizing movie files

The final movie library is stored under:

```text
/media/movies
```

A typical structure is:

```text
/media/movies/
└── Movie Name (Year)/
    └── Movie Name (Year).mkv
```

This structure improves metadata matching inside Jellyfin.

---

# Prowlarr

Prowlarr provides centralized indexer management.

Instead of configuring indexers independently inside Sonarr and Radarr, they can be managed centrally through Prowlarr.

Conceptually:

```text
Indexer Sources
      │
      ▼
   Prowlarr
      │
      ├──► Sonarr
      └──► Radarr
```

This reduces duplicated configuration and makes indexer management easier to maintain.

---

# qBittorrent

qBittorrent acts as the download client for the media automation stack.

Sonarr and Radarr send download jobs to qBittorrent through its API.

The workflow is:

```text
Sonarr / Radarr
      │
      ▼
  qBittorrent
      │
      ▼
   Downloads
      │
      ▼
Sonarr / Radarr Import
```

Once a download is complete, Sonarr or Radarr imports the file into the appropriate media library.

---

# Download VPN Separation

The remote-access VPN architecture is intentionally separated from downloader privacy.

Tailscale and WireGuard are used for:

```text
Remote Device
      │
      ▼
Secure Access
      │
      ▼
HomeLab
```

They are not intended to route qBittorrent traffic.

The planned downloader architecture is:

```text
qBittorrent
     │
     ▼
VPN Gateway
     │
     ▼
Internet
```

A future implementation may use **Gluetun** so that only the downloader traffic is routed through a commercial VPN tunnel.

This would prevent unrelated services such as:

- Jellyfin
- Home Assistant
- Prometheus
- Grafana
- Nginx Proxy Manager

from being unnecessarily routed through the download VPN.

---

# Bazarr

Bazarr provides automated subtitle management.

It integrates with the existing Sonarr and Radarr libraries and can search for subtitles for:

- TV episodes
- Movies

Its purpose is to extend the media automation workflow without modifying the core download pipeline.

Conceptually:

```text
Sonarr / Radarr Libraries
          │
          ▼
        Bazarr
          │
          ▼
      Subtitles
```

---

# Dispatcharr

Dispatcharr is used for IPTV and live TV management.

This service is separate from the normal movie and TV automation workflow.

Its purpose includes managing:

- IPTV sources
- Channel organization
- Live TV streams

This expands the media environment beyond static movie and episode libraries.

---

# Shared Storage

The media stack depends on the shared external storage mounted on the Proxmox host.

The physical disk is mounted at:

```text
/mnt/media
```

and exposed to the relevant workloads.

The final media paths are:

```text
/mnt/media/movies
/mnt/media/tv
```

Inside application environments these are exposed consistently as:

```text
/media/movies
/media/tv
```

Path consistency is important because multiple applications interact with the same files.

---

# Why Path Consistency Matters

An early source of problems in the media stack was inconsistent container paths.

For example, if one service sees:

```text
/downloads/movie.mkv
```

while another service expects:

```text
/data/downloads/movie.mkv
```

the applications may refer to the same physical file using different paths.

This can cause:

- Failed imports
- Missing file errors
- Remote path mapping issues
- Duplicate files
- Broken hardlinks or moves

The final design therefore attempts to keep path mappings predictable and consistent between containers.

---

# Media Data Flow

The complete simplified data flow is:

```text
Request
   │
   ▼
Jellyseerr
   │
   ▼
Sonarr / Radarr
   │
   ▼
Prowlarr
   │
   ▼
Indexer
   │
   ▼
Sonarr / Radarr
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
   ├──► /media/tv
   │
   └──► /media/movies
              │
              ▼
           Jellyfin
```

This divides the workflow into:

1. Request
2. Discovery
3. Download
4. Import
5. Organization
6. Playback

---

# Docker Networking

Docker provides its own virtual networking layer inside the LXC container.

Containers can communicate through Docker networks without requiring every application to be directly exposed to the LAN.

Conceptually:

```text
Home LAN
   │
   ▼
Docker LXC
   │
   ▼
Docker Bridge Network
   │
   ├── Sonarr
   ├── Radarr
   ├── Prowlarr
   ├── Bazarr
   ├── qBittorrent
   ├── Jellyseerr
   └── Monitoring Services
```

Only services that need to be accessed externally require ports to be published from Docker.

---

# Future Docker Network Segmentation

The current Docker environment can be improved further by creating dedicated application networks.

For example:

```text
proxy
media
downloads
monitoring
```

A possible future design could look like:

```text
                    Nginx Proxy Manager
                          │
                        proxy
                          │
            ┌─────────────┼─────────────┐
            │             │             │
         Jellyseerr    Grafana       Uptime Kuma

Sonarr ───────┐
Radarr ───────┼──── media
Prowlarr ─────┤
Bazarr ───────┘

Sonarr ───────┐
Radarr ───────┼──── downloads ─── qBittorrent
              │

Prometheus ───┐
Grafana ──────┼──── monitoring
Netdata ──────┘
```

Containers would only join the networks they actually require.

This would reduce unnecessary east-west communication between application containers.

---

# Nginx Proxy Manager

Nginx Proxy Manager acts as the reverse-proxy layer for selected internal applications.

Without a reverse proxy, applications are typically accessed using:

```text
http://192.168.1.x:PORT
```

The reverse proxy enables cleaner hostname-based access.

Conceptually:

```text
Client
   │
   ▼
Internal DNS
   │
   ▼
Nginx Proxy Manager
   │
   ├──► Jellyseerr
   ├──► Grafana
   ├──► Uptime Kuma
   └──► Other Services
```

This provides a centralized point for:

- Reverse proxy routing
- Hostname management
- TLS termination
- Certificate management

Nginx Proxy Manager is not treated as a replacement for firewalling or application authentication.

---

# Monitoring Services on Docker

The Docker LXC also hosts several components of the monitoring stack.

These include:

- Prometheus
- Grafana
- Uptime Kuma
- Netdata
- Glances
- What's Up Docker

These services are documented in greater depth in:

[Monitoring & Observability](monitoring.md)

The key distinction is that the Docker host runs the applications, while monitoring is treated as a separate logical infrastructure function.

---

# What's Up Docker

What's Up Docker monitors deployed Docker images and identifies available updates.

This avoids manually checking each service.

The workflow is:

```text
Running Containers
       │
       ▼
What's Up Docker
       │
       ▼
Registry Version Check
       │
       ▼
Update Available?
```

The tool is used for **visibility**, not blind automatic updating.

Updates can first be reviewed and then applied according to the maintenance process.

---

# Speedtest Tracker

Speedtest Tracker monitors WAN performance over time.

It records metrics such as:

- Download throughput
- Upload throughput
- Latency

This helps distinguish application-level problems from Internet connectivity issues.

For example:

```text
Slow Jellyfin Access
       │
       ├── LAN issue?
       ├── Host issue?
       └── WAN issue?
```

Speedtest history provides another data point during troubleshooting.

---

# Persistent Configuration

Docker services require configuration to survive container recreation.

Configuration directories and volumes are therefore stored persistently outside the ephemeral container filesystem.

A typical Compose pattern is:

```yaml
services:
  application:
    volumes:
      - ./config/application:/config
```

This separates:

```text
Container Image
      │
      └── Disposable

Configuration / Data
      │
      └── Persistent
```

As a result, containers can be recreated or upgraded without losing application configuration.

---

# Docker Compose

Docker Compose is used to define services declaratively.

A Compose file can define:

- Image
- Container name
- Environment variables
- Volumes
- Networks
- Published ports
- Restart policy
- Dependencies

Example sanitized structure:

```yaml
services:

  example-service:
    image: vendor/application:latest
    container_name: example-service

    restart: unless-stopped

    environment:
      - TZ=Europe/Rome

    volumes:
      - ./config/example:/config

    ports:
      - "8080:8080"
```

The public repository should contain sanitized examples rather than production credentials.

---

# Secrets Management

Docker configurations may contain sensitive information such as:

- API keys
- Passwords
- Tokens
- VPN credentials
- Application secrets

These should not be stored directly in public Compose files.

A preferred pattern is:

```yaml
environment:
  API_KEY: ${API_KEY}
```

with values stored separately in:

```text
.env
```

The real `.env` file should be excluded from Git.

A public example can be provided as:

```text
.env.example
```

containing placeholders:

```env
API_KEY=REPLACE_ME
PASSWORD=REPLACE_ME
```

This allows the infrastructure configuration to remain reproducible without exposing credentials.

---

# Container Update Strategy

Container updates are treated as infrastructure changes rather than automatically applied without review.

The process is:

```text
Update detected
      │
      ▼
Review release/change
      │
      ▼
Backup configuration
      │
      ▼
Pull new image
      │
      ▼
Recreate container
      │
      ▼
Validate service
```

Typical commands include:

```bash
docker compose pull
docker compose up -d
```

After an update:

```bash
docker compose ps
docker compose logs
```

can be used to confirm service health.

---

# Restart Policies

Long-running services use appropriate Docker restart policies.

A common configuration is:

```yaml
restart: unless-stopped
```

This allows applications to recover automatically after:

- Docker daemon restart
- LXC restart
- Proxmox reboot

while still respecting deliberate manual shutdowns.

---

# Security Considerations

The Docker stack creates several attack surfaces.

Potential risks include:

- Vulnerable container images
- Exposed Web UIs
- Weak credentials
- Excessive container privileges
- Mounted host directories
- Docker socket exposure
- Untrusted downloaded content
- Excessive network connectivity
- Outdated dependencies

Security improvements therefore focus on minimizing privileges and exposure.

---

## Avoiding Privileged Containers

Applications should run without privileged Docker mode unless strictly required.

Avoid where possible:

```yaml
privileged: true
```

because this significantly increases host access.

---

## Docker Socket

Access to:

```text
/var/run/docker.sock
```

effectively provides powerful control over the Docker daemon.

Any container receiving access to the Docker socket must therefore be treated as highly trusted.

Services that require Docker API visibility should receive only the minimum access necessary.

---

## Port Exposure

Not every Docker application needs to expose its interface directly to the LAN.

Where possible:

```text
Client
   │
   ▼
Reverse Proxy
   │
   ▼
Application
```

can reduce the number of directly published application endpoints.

However, reverse proxying does not remove the need for authentication or firewall controls.

---

## Container Images

Images should be obtained from trusted or well-maintained sources.

Updates should be reviewed because a container image is part of the software supply chain.

Monitoring image changes through What's Up Docker helps provide visibility into this process.

---

# Backup Considerations

Application containers themselves are generally disposable.

The important data to protect is:

- Docker Compose configuration
- Application configuration
- Databases
- Persistent volumes
- Environment configuration
- Custom scripts

The backup strategy should therefore focus on persistent state rather than container layers.

A future improvement is to automate configuration backups before Docker updates.

---

# Challenges and Lessons Learned

## Docker Inside LXC

The main challenge was running Docker under an LXC security model.

AppArmor restrictions initially prevented Docker from functioning correctly.

The solution required enabling nesting and relaxing some confinement.

This demonstrated an important infrastructure principle:

> Compatibility fixes can introduce security trade-offs.

---

## Media Path Consistency

Consistent storage paths across applications are critical.

Misaligned path mappings caused import and library problems.

The solution was to standardize application-visible paths around:

```text
/media/movies
/media/tv
```

where possible.

---

## Service Dependencies

The media environment contains multiple API-dependent services.

A single service failure can affect later stages of the pipeline.

For example:

```text
Prowlarr failure
      │
      ▼
Sonarr/Radarr cannot search
      │
      ▼
No download requested
```

Monitoring therefore needs to check application availability as well as host resource usage.

---

## Update Management

Running many containers makes software maintenance more complex.

What's Up Docker was added to provide visibility into available updates without forcing uncontrolled automatic upgrades.

---

# Future Improvements

Planned improvements for the Docker environment include:

- Gluetun VPN isolation for qBittorrent
- Dedicated Docker networks by service role
- Reduced direct port exposure
- More consistent health checks
- Automated configuration backups
- Improved secrets management
- Version-controlled sanitized Compose files
- Pinning container versions where appropriate
- Reviewing container capabilities
- Minimizing Docker socket exposure
- Automated vulnerability scanning of images
- Better monitoring of container restarts
- Alerting for failed health checks
- Potential migration of high-risk workloads to a dedicated VM

---

# Key Takeaways

The Docker stack demonstrates practical experience with:

- Docker
- Docker Compose
- Nested containerization
- Linux LXC containers
- Application deployment
- Persistent volumes
- Docker networking
- Reverse proxying
- Media automation
- API integrations
- Monitoring
- Update management
- Security trade-offs
- Troubleshooting distributed services

The goal is not simply to run a collection of Docker containers, but to structure them as a maintainable application platform with clear dependencies, persistent storage, monitoring, and documented security considerations.

---

## Related Documentation

- [Architecture](architecture.md)
- [Networking](networking.md)
- [Monitoring & Observability](monitoring.md)
- [Jellyfin](jellyfin.md)
- [Cybersecurity](cybersecurity.md)
- [Backup & Recovery](backup-recovery.md)
