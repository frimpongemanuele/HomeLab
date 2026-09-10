# Networking

## Overview

Networking is a core component of my HomeLab because nearly every service depends on reliable communication between the Proxmox host, virtualized workloads, Docker applications, smart-home devices, client devices, and remote systems.

The current network was designed around a practical home-network environment and has gradually evolved as additional infrastructure services were introduced.

The main networking objectives are:

- Provide stable connectivity to self-hosted services
- Avoid unnecessary Internet-facing services
- Provide secure remote access
- Centralize DNS filtering
- Provide consistent internal service access
- Monitor network availability and WAN performance
- Support reverse proxying for internal applications
- Prepare the infrastructure for future VLAN segmentation

The current environment still operates primarily on a **single trusted LAN**, which is an acknowledged limitation and one of the main areas planned for future improvement.

---

# Current Network Architecture

![Current HomeLab Network Architecture](../diagrams/exported/network-architecture.svg)

The current HomeLab operates primarily on a single `192.168.1.0/24` LAN behind the TIM HUB+ router.

The Proxmox host acts as the main infrastructure node and connects the virtualized workloads to the physical network through a Linux bridge.

At the current stage, client devices, IoT devices, infrastructure services, and experimental workloads still share the same Layer-2 network. This keeps the environment simple to operate, but it also represents one of the main areas planned for future security improvement through VLAN-based segmentation.

---

# Physical Network

## Internet Gateway

The primary Internet gateway is a **TIM HUB+** router.

It currently provides:

- Internet connectivity
- NAT
- DHCP
- LAN switching
- Wi-Fi connectivity
- Default gateway functionality

The router is available internally at:

```text
192.168.1.1
```

The primary HomeLab network currently uses:

```text
192.168.1.0/24
```

This provides up to 254 usable IPv4 addresses within the local subnet.

---

# Proxmox Networking

The Proxmox server is the central compute node of the HomeLab.

Current management address:

```text
192.168.1.69
```

Proxmox uses a Linux bridge to connect virtual machines and LXC containers to the physical network interface.

Conceptually:

```text
Physical NIC
    │
    ▼
Proxmox Linux Bridge
    │
    ├── VM 100
    ├── LXC 101
    ├── LXC 102
    ├── LXC 103
    ├── LXC 104
    ├── LXC 105
    └── LXC 106
```

This allows virtual workloads to behave like independent systems connected directly to the LAN.

Each workload can therefore have its own:

- IP address
- Network configuration
- Listening services
- Firewall policy
- DNS configuration

while sharing the physical network interface of the Proxmox host.

---

# Current Virtualized Network Services

The current Proxmox environment contains the following main workloads:

| ID | System | Role |
|---|---|---|
| VM 100 | Home Assistant OS | Smart-home platform |
| LXC 101 | Jellyfin | Media server |
| LXC 102 | Docker | Application container host |
| LXC 103 | Tailscale | Secure remote-access networking |
| LXC 104 | Homepage | Central dashboard |
| LXC 105 | Pi-hole | DNS filtering |
| LXC 106 | WebApp | Development/security testing |

These systems currently communicate through the same home LAN unless application-level restrictions are applied.

---

# DNS Architecture

## Pi-hole

Pi-hole runs inside a dedicated Proxmox LXC container.

Its primary purpose is to provide DNS-level filtering for the HomeLab and selected network clients.

Pi-hole can block requests to known:

- Advertising domains
- Tracking domains
- Telemetry endpoints
- Unwanted services
- Custom blocklisted domains

Conceptually:

```text
Client
   │
   ▼
Pi-hole
   │
   ├── Blocked domain
   │       │
   │       └──► Request denied
   │
   └── Allowed domain
           │
           ▼
      Upstream DNS
```

This provides both filtering and visibility into DNS activity.

---

## Why Pi-hole Runs in a Dedicated LXC

Pi-hole could have been deployed inside the existing Docker environment.

Instead, it runs in its own lightweight LXC.

This was an intentional architectural choice.

DNS is an infrastructure dependency. If Pi-hole were hosted inside the main Docker LXC, restarting or troubleshooting Docker could also interrupt DNS resolution.

The dedicated container therefore reduces dependency between:

```text
DNS Infrastructure
        and
Application Infrastructure
```

This improves operational resilience.

---

# DNS as a Security Control

Pi-hole is primarily a DNS filtering platform rather than a complete security product.

However, DNS filtering can contribute to defense-in-depth by blocking known or unwanted destinations before applications connect to them.

DNS logs can also provide useful indicators during troubleshooting or security investigations.

Examples include:

```text
Unexpected domain queries
Repeated requests to unknown domains
High-volume DNS activity
Devices contacting unexpected external services
```

Pi-hole therefore contributes both to **network hygiene** and **network visibility**.

It is not treated as a replacement for:

- Firewalls
- IDS/IPS
- Endpoint security
- Network segmentation

---

# Internal Service Access

Most HomeLab services are accessed directly through private IP addresses and application ports.

Examples include:

```text
Proxmox
https://<proxmox-ip>:8006

Home Assistant
http://<home-assistant-ip>:8123

Jellyfin
http://192.168.1.66:8096

Homepage
http://<homepage-ip>:3000
```

Management interfaces remain on the private network rather than being intentionally exposed directly to the public Internet.

---

# Reverse Proxy

The Docker environment includes **Nginx Proxy Manager**.

Its purpose is to provide a centralized reverse-proxy layer for selected internal web services.

Without a reverse proxy, applications are normally accessed using combinations such as:

```text
192.168.1.x:PORT
```

A reverse proxy makes it possible to introduce cleaner internal service names:

```text
service.internal.domain
```

Conceptually:

```text
Client
   │
   ▼
DNS
   │
   ▼
Nginx Proxy Manager
   │
   ├──► Application A
   ├──► Application B
   ├──► Application C
   └──► Application D
```

This centralizes application routing while allowing the backend services to continue listening on their own ports.

---

# Reverse Proxy Security

A reverse proxy improves service organization but should not be confused with a firewall.

Nginx Proxy Manager can provide capabilities such as:

- Centralized HTTP routing
- TLS termination
- Certificate management
- Hostname-based service access
- Additional access controls

However:

> Reverse proxying does not automatically make a service secure.

Backend services still require:

- Authentication
- Updates
- Secure configuration
- Appropriate network access controls

The current reverse proxy is primarily used as an internal infrastructure component.

---

# Remote Access

Remote access is intentionally implemented through VPN technologies rather than conventional public port forwarding.

The HomeLab currently uses:

- Tailscale
- WireGuard

These serve related but distinct remote-access purposes.

---

# Tailscale

Tailscale is the primary remote-access solution for the HomeLab infrastructure.

A dedicated Proxmox LXC is used for the Tailscale service.

Tailscale creates an encrypted overlay network based on WireGuard.

Conceptually:

```text
Remote Device
      │
      ▼
   Internet
      │
      ▼
Encrypted Tailscale Tunnel
      │
      ▼
HomeLab Network
      │
      ├── Proxmox
      ├── Homepage
      ├── Home Assistant
      ├── Jellyfin
      └── Internal Services
```

This allows remote access without exposing management interfaces directly to the Internet.

---

# Why Tailscale Instead of Port Forwarding

Traditional remote access could be implemented by forwarding router ports such as:

```text
8006 → Proxmox
8123 → Home Assistant
8096 → Jellyfin
```

This would expose those services directly to Internet-originated traffic.

Instead, the preferred architecture is:

```text
Internet
    │
    X
No direct management ports
    │
    ▼
Encrypted VPN
    │
    ▼
Private HomeLab services
```

This significantly reduces the publicly reachable attack surface.

---

# WireGuard

WireGuard is also used through Home Assistant for secure remote access.

It provides encrypted connectivity back into the home network.

This was originally configured to allow remote access to Home Assistant and other internal resources.

WireGuard and Tailscale therefore both provide secure tunneling capabilities, although Tailscale simplifies peer discovery, identity, and overlay networking.

---

# Remote-Access VPN vs Download VPN

An important architectural distinction is maintained between:

```text
Remote Access VPN
```

and:

```text
Outbound Privacy VPN
```

Tailscale and the Home Assistant WireGuard deployment are used to **enter the HomeLab securely from outside**.

They are not designed as the privacy layer for download traffic.

The planned media-download architecture instead isolates downloader traffic separately.

Conceptually:

```text
Remote Access:

Laptop / Phone
      │
      ▼
Tailscale / WireGuard
      │
      ▼
HomeLab


Download Privacy:

qBittorrent
      │
      ▼
VPN Gateway
      │
      ▼
Internet
```

Keeping these two functions separate avoids unnecessary routing complexity.

---

# Docker Networking

The Docker host runs multiple application containers.

Docker provides its own virtual networking layer on top of the LXC network stack.

The architecture is therefore:

```text
Physical Ethernet
       │
       ▼
Proxmox Bridge
       │
       ▼
Docker LXC
       │
       ▼
Docker Networks
       │
       ├── Sonarr
       ├── Radarr
       ├── Prowlarr
       ├── qBittorrent
       ├── Bazarr
       ├── Jellyseerr
       ├── Dispatcharr
       ├── Nginx Proxy Manager
       ├── Prometheus
       ├── Grafana
       └── Other Services
```

Docker bridge networks provide application-level isolation and service discovery between containers.

This is useful, but Docker network isolation should not be confused with physical network segmentation or VLAN isolation.

---

# Media Stack Network Flow

The media automation stack communicates internally across several services.

A simplified logical flow is:

```text
Jellyseerr
     │
     ▼
Sonarr / Radarr
     │
     ├────► Prowlarr
     │
     ▼
qBittorrent
     │
     ▼
Media Storage
     │
     ▼
Jellyfin
```

These services require controlled connectivity between APIs while only selected user-facing interfaces need to be accessed directly by clients.

This architecture provides an opportunity for future Docker network segmentation.

For example, containers could be separated into networks such as:

```text
frontend
media
downloads
monitoring
proxy
```

with services connected only to the networks they require.

---

# Samba / SMB Networking

The external media disk mounted on the Proxmox host is also shared to Windows systems through Samba.

The share is available from the Proxmox host rather than the Jellyfin LXC.

Conceptually:

```text
Windows PC
    │
    │ SMB
    ▼
Proxmox Host
    │
    ▼
/mnt/media
    │
    ├── movies
    ├── tv
    └── proxmox-backups
```

A dedicated Samba account is used instead of anonymous guest access.

SMB1 was disabled after troubleshooting because it is obsolete and insecure.

Modern SMB versions provide the required interoperability without enabling the legacy protocol.

---

# Network Monitoring

The HomeLab includes several tools that provide network-related visibility.

These include:

### Uptime Kuma

Monitors whether internal services and endpoints are reachable.

### Prometheus

Collects network-related metrics from monitored infrastructure.

### Grafana

Visualizes historical network metrics.

### Netdata

Provides detailed real-time network interface activity.

### Glances

Provides lightweight network utilization visibility.

### Speedtest Tracker

Records WAN performance including:

- Download speed
- Upload speed
- Latency

### Pi-hole

Provides DNS query visibility.

Together these tools provide visibility across multiple layers:

```text
DNS
 │
Pi-hole

Service Availability
 │
Uptime Kuma

Host / Interface Metrics
 │
Prometheus / Netdata / Glances

Historical Visualization
 │
Grafana

WAN Performance
 │
Speedtest Tracker
```

Detailed observability architecture is documented in:

[Monitoring & Observability](monitoring.md)

---

# Current Security Model

The current network follows a **low-exposure** rather than a fully segmented security model.

Implemented controls include:

- Private RFC1918 addressing
- NAT at the Internet gateway
- No intentional direct exposure of administrative interfaces
- VPN-based remote access
- DNS filtering
- Dedicated Pi-hole infrastructure
- Authenticated SMB access
- SMB1 disabled
- Workload isolation through VMs and LXC containers
- Docker application isolation
- Reverse proxy for selected web applications
- Dedicated WebApp testing environment

These controls reduce risk, but they do not provide complete isolation between different classes of devices.

---

# Current Limitation: Flat LAN

The most important current networking limitation is that the HomeLab still primarily operates on a single LAN:

```text
192.168.1.0/24
```

This means systems with very different trust levels can potentially exist within the same Layer-2 network.

Examples include:

```text
Management systems
Servers
Personal computers
Phones
IoT devices
Smart-home devices
Experimental workloads
```

Virtualization isolates workloads at the compute level, but does not automatically provide strong network segmentation.

For example:

```text
LXC 106 WebApp
```

is isolated from the Proxmox host as a container, but if both systems share the same unrestricted LAN, network-level communication may still be possible.

This distinction is important:

> **Virtualization isolation is not the same as network segmentation.**

---

# Planned VLAN Architecture

The next major networking improvement is VLAN-based segmentation.

The proposed architecture separates devices and services according to trust level and purpose.

```text
                         Internet
                            │
                            ▼
                     Router / Firewall
                            │
            ┌───────────────┼────────────────┐
            │               │                │
            ▼               ▼                ▼
       Management        Services           IoT
          VLAN             VLAN             VLAN
            │               │                │
        Proxmox          Jellyfin        Smart Plugs
        Grafana          Apps            Sensors
        Homepage         Media           IoT Devices

            ┌─────────────────────────────────┐
            │                                 │
            ▼                                 ▼
         Lab VLAN                         Guest VLAN
            │                                 │
        WebApp Lab                        Guest Wi-Fi
        Security Labs                     Internet Only
```

---

# Proposed Trust Zones

A future network could use zones such as:

| Zone | Purpose | Example Systems |
|---|---|---|
| Management | Infrastructure administration | Proxmox, monitoring |
| Services | User-facing internal services | Jellyfin, media apps |
| IoT | Smart-home devices | Plugs, sensors, lights |
| Lab | Experimental workloads | WebApp, security testing |
| Guest | Untrusted personal devices | Guest phones/laptops |

The exact VLAN IDs and subnets have intentionally not been assigned yet because the design has not been implemented.

This prevents documentation from presenting planned configuration as existing infrastructure.

---

# Proposed Firewall Policy

The future VLAN architecture would follow a **default-deny inter-zone model**.

Instead of allowing unrestricted communication between networks:

```text
ALLOW everything
BLOCK exceptions
```

the preferred model is:

```text
BLOCK by default
ALLOW required flows
```

Example policy:

| Source | Destination | Policy |
|---|---|---|
| Management | All internal zones | Allow required administration |
| Services | Management | Deny by default |
| IoT | Management | Deny |
| IoT | Internet | Allow required outbound |
| IoT | Home Assistant | Allow required protocols |
| Lab | Management | Deny |
| Lab | Services | Deny by default |
| Guest | Internal networks | Deny |
| Guest | Internet | Allow |

Specific exceptions would then be introduced only when required.

---

# IoT Segmentation

IoT devices represent one of the strongest reasons for implementing VLANs.

The Home Assistant environment includes devices such as:

- Smart lights
- Smart plugs
- Sensors
- Zigbee devices
- Wi-Fi IoT devices

Many consumer IoT devices receive limited long-term security support and may communicate with external cloud infrastructure.

The future objective is therefore:

```text
IoT Device
     │
     ├────► Home Assistant     ALLOW
     │
     ├────► Required Internet  ALLOW
     │
     └────► Management LAN     DENY
```

This reduces the impact of a compromised IoT device.

---

# Lab Network Isolation

The WebApp LXC is specifically intended for experimentation.

Potential workloads may include:

- Custom web applications
- Development frameworks
- Test APIs
- Intentionally vulnerable applications
- OWASP exercises
- Security testing targets

Because these workloads may have a higher probability of vulnerabilities, they should eventually operate inside a dedicated Lab VLAN.

The desired security boundary is:

```text
Lab
 │
 ├──► Internet             Controlled
 │
 ├──► Other Lab Targets    Allowed as required
 │
 ├──X► Management          Denied
 │
 └──X► Sensitive Services  Denied
```

This will allow security experimentation without unnecessarily increasing the risk to the primary HomeLab infrastructure.

---

# Network Attack Surface

The network attack surface currently includes several categories.

## Management Interfaces

Examples:

- Proxmox
- Grafana
- Pi-hole administration
- Nginx Proxy Manager
- Homepage

Compromise of management interfaces could provide visibility or control over other infrastructure components.

---

## Application Services

Examples:

- Jellyfin
- Home Assistant
- Jellyseerr
- Sonarr
- Radarr

These applications expose HTTP interfaces and APIs that require regular patching and appropriate authentication.

---

## Infrastructure Protocols

Examples:

- DNS
- SMB
- HTTP/HTTPS
- VPN
- Docker networking

Misconfiguration of infrastructure protocols could expose sensitive data or increase lateral-movement opportunities.

---

## IoT Devices

IoT devices represent a different security boundary because firmware, update mechanisms, and cloud dependencies are often outside the administrator's control.

---

## Experimental Systems

The WebApp lab intentionally introduces potentially unstable or vulnerable workloads.

This makes network isolation particularly important.

---

# Network Troubleshooting Workflow

The monitoring stack provides several layers for diagnosing network issues.

A typical workflow is:

```text
Service unavailable
        │
        ▼
Check Homepage
        │
        ▼
Check Uptime Kuma
        │
        ▼
Can the host be reached?
    │             │
   Yes            No
    │             │
    ▼             ▼
Application    Network
Investigation Investigation
                  │
                  ▼
            Ping / DNS / Route
                  │
                  ▼
          Netdata / Prometheus
                  │
                  ▼
            Router / Firewall
```

Useful Linux tools include:

```bash
ip addr
ip route
ss -lntp
ping
traceroute
dig
nslookup
curl
```

These allow troubleshooting at different layers of the network stack.

---

# Lessons Learned

Building the HomeLab networking environment reinforced several practical lessons.

### IP conflicts can create misleading failures

An earlier Jellyfin connectivity problem was eventually traced to an old server still operating on the network.

This demonstrated that application failures can sometimes originate from basic network addressing problems rather than the application itself.

---

### Infrastructure dependencies should be separated

Running Pi-hole in a dedicated LXC reduces the dependency between DNS and the Docker application environment.

---

### Remote access does not require public exposure

Tailscale and WireGuard allow internal services to remain private while still being accessible remotely.

---

### Docker networks are not VLANs

Container networks provide useful application isolation but do not replace segmentation at the LAN/firewall level.

---

### DNS is valuable security telemetry

DNS logs can reveal unexpected communication patterns even when full packet inspection is unavailable.

---

### Legacy protocols should not be enabled permanently for troubleshooting

SMB1 was temporarily considered during troubleshooting but was not required and should remain disabled.

---

### Network architecture should reflect trust boundaries

A flat network is simple to operate, but becomes increasingly inappropriate as infrastructure, IoT devices, and experimental systems grow.

---

# Future Improvements

The main planned networking improvements are:

- VLAN segmentation
- Dedicated Management VLAN
- Dedicated Services VLAN
- Dedicated IoT VLAN
- Dedicated Lab VLAN
- Guest network isolation
- Inter-VLAN firewall policies
- Default-deny communication between trust zones
- Improved internal DNS
- Expanded internal HTTPS
- More granular Docker networks
- VPN isolation for downloader traffic
- DNS monitoring integration with Grafana
- Network intrusion detection
- Suricata deployment
- Firewall log collection
- Centralized network logging
- Configuration automation
- Network documentation as code

A future security-oriented architecture could eventually resemble:

```text
                       Internet
                          │
                          ▼
                   Router / Firewall
                          │
             ┌────────────┼────────────┐
             │            │            │
        Management     Services       IoT
             │            │            │
             └──────┬─────┴─────┬──────┘
                    │           │
                    ▼           ▼
                  Lab         Guest
                    │
                    ▼
              Security Testing

              Network Telemetry
                    │
          ┌─────────┼─────────┐
          │         │         │
          ▼         ▼         ▼
       Pi-hole   Suricata   Firewall
          │         │         │
          └─────────┼─────────┘
                    ▼
             Central Logging
                    │
                    ▼
               SIEM / Wazuh
```

This would evolve the current HomeLab from a primarily functional home network into a segmented environment capable of supporting more realistic cybersecurity monitoring and testing.

---

# Key Takeaways

The networking architecture demonstrates practical experience with:

- TCP/IP networking
- Private IPv4 addressing
- Linux bridges
- Proxmox networking
- Virtual machine networking
- LXC networking
- Docker networking
- DNS infrastructure
- Pi-hole
- Reverse proxies
- Nginx Proxy Manager
- SMB file sharing
- VPN technologies
- Tailscale
- WireGuard
- Service monitoring
- Network troubleshooting
- Security architecture
- Trust boundaries
- VLAN planning
- Firewall policy design

The current environment intentionally balances simplicity with security.

Rather than presenting the network as fully segmented when it is not, the architecture documents both the controls currently implemented and the limitations that will guide the next stage of the HomeLab.

---

## Related Documentation

- [Architecture](architecture.md)
- [Cybersecurity](cybersecurity.md)
- [Monitoring & Observability](monitoring.md)
- [Docker & Media Stack](docker-media-stack.md)
- [Home Assistant](home-assistant.md)
- [Lab Environment](lab-environment.md)
