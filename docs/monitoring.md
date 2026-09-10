# Monitoring & Observability

## Overview

Monitoring and observability are core components of my HomeLab.

As the environment expanded from a small Proxmox installation into multiple VMs, LXC containers, Docker services, networking components, and smart-home workloads, relying only on individual application dashboards was no longer sufficient.

I therefore implemented a layered monitoring architecture designed to answer several different operational questions:

- Are my services online?
- Is the Proxmox host healthy?
- Are CPU, memory, disk, or network resources becoming constrained?
- How are Docker workloads behaving over time?
- Are software updates available?
- Is my Internet connection performing normally?
- Can infrastructure metrics be analyzed historically?
- Can the overall state of the HomeLab be checked from one place?

Instead of relying on a single monitoring product, the environment uses multiple specialized tools.

---

## Monitoring Architecture

![Monitoring Architecture](../diagrams/exported/monitoring-architecture.svg)

The monitoring stack follows a layered model:

- **Infrastructure and services** generate metrics and health data.
- **Prometheus** collects and stores time-series metrics.
- **Grafana** visualizes historical metrics and trends.
- **Uptime Kuma** monitors service availability.
- **Netdata** and **Glances** provide real-time system visibility.
- **What's Up Docker** tracks container image updates.
- **Speedtest Tracker** records WAN performance.
- **Homepage** aggregates operational status into a single dashboard.

This architecture separates **metrics collection**, **visualization**, **availability monitoring**, and **operational dashboards** rather than forcing one application to perform every role.

---

# Monitoring Stack

The current monitoring environment includes:

| Tool | Primary Role |
|---|---|
| Prometheus | Time-series metrics collection |
| Grafana | Metrics visualization and dashboards |
| Uptime Kuma | Service availability monitoring |
| Netdata | Detailed real-time system monitoring |
| Glances | Lightweight live system metrics |
| What's Up Docker | Docker image update monitoring |
| Speedtest Tracker | Internet performance history |
| Homepage | Central operational dashboard |

These tools intentionally overlap in some areas.

The objective is not simply redundancy. Each tool provides a different level of visibility into the infrastructure.

---

# Prometheus

## Purpose

Prometheus provides the primary **metrics collection and time-series database** for the HomeLab.

While tools such as Netdata provide excellent real-time visibility, Prometheus allows infrastructure metrics to be stored and queried historically.

This makes it possible to analyze questions such as:

- How has CPU utilization changed over time?
- Is memory usage gradually increasing?
- Are disk I/O patterns abnormal?
- Is network traffic increasing?
- Did resource usage change after deploying a new service?
- Was a performance issue temporary or persistent?

Prometheus therefore forms the foundation of the long-term observability stack.

---

## Metrics Collection Model

Prometheus follows a pull-based architecture.

Instead of infrastructure components continuously sending metrics to Prometheus, Prometheus periodically retrieves metrics from configured endpoints.

Conceptually:

```text
Proxmox / Docker / Linux
          │
          ▼
       Exporters
          │
          ▼
     /metrics endpoint
          │
          ▼
      Prometheus
```

Targets are configured inside the Prometheus configuration file.

A simplified configuration looks like:

```yaml
scrape_configs:

  - job_name: "docker-host"
    static_configs:
      - targets:
          - "HOST:PORT"

  - job_name: "proxmox"
    static_configs:
      - targets:
          - "EXPORTER:PORT"
```

Real IP addresses, credentials, API tokens, and secrets are intentionally excluded from the public repository.

---

# Proxmox Monitoring

Monitoring the Proxmox host is particularly important because it represents the central compute platform for almost every HomeLab workload.

Metrics collected from the virtualization environment can include:

- CPU utilization
- Memory utilization
- Storage utilization
- Network traffic
- VM state
- LXC state
- Node health
- Resource allocation

This provides visibility beyond what is available from individual applications.

---

## Dedicated Proxmox API Access

Monitoring systems should not require unrestricted administrative credentials.

Where API access is required, the preferred design is to use a **dedicated read-only Proxmox API user/token** with the minimum permissions necessary to retrieve metrics.

Conceptually:

```text
Prometheus
    │
    ▼
Proxmox Exporter
    │
    ▼
Read-Only API Token
    │
    ▼
Proxmox API
```

A restricted monitoring identity reduces the impact if monitoring credentials are ever exposed.

This follows the principle of:

> **Least privilege — monitoring systems should be able to observe infrastructure, not administer it.**

---

# Docker Monitoring

The Docker LXC hosts a significant portion of the HomeLab application stack.

Monitoring Docker is therefore important because multiple services share the same underlying compute resources.

Examples include:

- Sonarr
- Radarr
- Prowlarr
- Bazarr
- qBittorrent
- Jellyseerr
- Dispatcharr
- Nginx Proxy Manager
- Monitoring services

Docker monitoring focuses on both:

### Host-level metrics

- CPU utilization
- Memory utilization
- Disk usage
- Disk I/O
- Network traffic

### Container-level metrics

- Container CPU usage
- Container memory usage
- Network activity
- Container availability
- Resource trends

This makes it possible to identify which workload is responsible for increased resource consumption rather than only seeing the total utilization of the Docker host.

---

# Grafana

## Purpose

Grafana provides the primary visualization layer for Prometheus metrics.

Prometheus is optimized for collecting and querying time-series data, while Grafana converts those metrics into human-readable dashboards.

The data flow is:

```text
Infrastructure
      │
      ▼
   Exporters
      │
      ▼
  Prometheus
      │
      ▼
    Grafana
      │
      ▼
   Dashboards
```

---

## Dashboard Design

Rather than creating one extremely large dashboard, the monitoring environment can be divided into focused dashboards.

Current and planned dashboard categories include:

### Proxmox Overview

Metrics such as:

- CPU utilization
- Memory usage
- Storage usage
- Network traffic
- VM/LXC status

### Docker Overview

Metrics such as:

- Docker host CPU
- Docker host memory
- Container CPU usage
- Container memory usage
- Network traffic
- Disk I/O

### Storage Monitoring

Metrics such as:

- Disk usage
- Available storage
- Read/write activity
- Disk I/O

### Network Monitoring

Metrics such as:

- Network throughput
- Interface utilization
- WAN performance
- Latency

The goal is to make dashboards useful for troubleshooting rather than simply displaying as many graphs as possible.

---

# Uptime Kuma

Prometheus answers:

> "How is the infrastructure performing?"

Uptime Kuma answers:

> "Is the service actually reachable?"

Uptime Kuma is used for active service-health monitoring.

The current environment monitors multiple internal services and provides a centralized overview of whether they are operational.

Examples include:

- Proxmox
- Home Assistant
- Jellyfin
- Homepage
- Pi-hole
- Docker-hosted applications
- Monitoring services

---

## Availability Monitoring

Uptime Kuma can perform checks such as:

```text
HTTP / HTTPS
TCP
Ping
DNS
```

This is useful because a system can have healthy CPU and memory metrics while an application running on it is unavailable.

For example:

```text
Host CPU: 15%
Memory: 50%
Network: Healthy

BUT

Jellyfin HTTP check: DOWN
```

Infrastructure metrics alone would not necessarily detect this application-level failure.

Combining Prometheus and Uptime Kuma therefore provides both **performance monitoring** and **availability monitoring**.

---

# Netdata

Netdata provides detailed real-time visibility into system performance.

It is particularly useful during troubleshooting because it exposes a large number of system metrics with minimal manual dashboard configuration.

Typical metrics include:

- CPU utilization
- Memory
- Swap
- Disk activity
- Disk I/O
- Network interfaces
- Processes
- Load averages
- Filesystem usage

Netdata complements Prometheus/Grafana.

Prometheus and Grafana are primarily used for historical analysis and customized dashboards, while Netdata is useful for immediate real-time investigation.

---

# Glances

Glances provides lightweight real-time system monitoring.

It gives a fast overview of:

- CPU
- Memory
- Load
- Disk
- Network
- Processes

Glances is useful when a complete Grafana or Netdata interface is unnecessary.

Conceptually:

```text
Grafana  → Historical analysis
Netdata  → Detailed real-time analysis
Glances  → Quick live overview
```

The tools overlap intentionally but serve different operational workflows.

---

# What's Up Docker

Containerized environments introduce another operational problem:

> How do I know when new Docker images are available?

**What's Up Docker (WUD)** monitors Docker containers and compares their currently deployed images with available upstream versions.

It provides visibility into:

- Containers being monitored
- Available image updates
- Image version changes

This helps maintain the Docker environment without blindly running updates across every service.

Updates are still reviewed before deployment because automatically updating infrastructure without validation can introduce compatibility problems.

This fits the broader HomeLab update strategy:

```text
Detect Update
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
      │
   ┌──┴──┐
   │     │
Success Failure
   │     │
   ▼     ▼
 Keep   Rollback
```

---

# Speedtest Tracker

Speedtest Tracker provides historical monitoring of the Internet connection.

Metrics include:

- Download speed
- Upload speed
- Latency

This makes it possible to distinguish between:

```text
Application problem
        vs.
LAN problem
        vs.
Internet connection problem
```

Historical measurements are particularly useful when diagnosing intermittent connectivity or ISP performance issues.

---

# Homepage as the Operational Dashboard

Homepage acts as the main entry point into the HomeLab.

It should not be confused with the monitoring backend itself.

Instead, Homepage acts as an **operational aggregation layer**.

The dashboard currently groups services into categories such as:

```text
Smart Home
Media
Infrastructure
Downloads
Network
Monitoring
```

The monitoring section exposes the status of tools including:

- Uptime Kuma
- Netdata
- Glances
- What's Up Docker
- Grafana
- Prometheus

Homepage therefore provides a fast answer to:

> "What is the overall state of my HomeLab right now?"

while Grafana, Prometheus, Netdata, and Uptime Kuma provide deeper diagnostic information.

---

# Observability Layers

The monitoring design can be summarized as several complementary layers.

| Layer | Tool | Question Answered |
|---|---|---|
| Operational Overview | Homepage | What is happening right now? |
| Availability | Uptime Kuma | Is the service reachable? |
| Metrics Collection | Prometheus | What metrics are being recorded? |
| Historical Visualization | Grafana | How has the system behaved over time? |
| Real-Time Diagnostics | Netdata | What is happening inside the system now? |
| Quick Diagnostics | Glances | What are the key system metrics? |
| Update Monitoring | What's Up Docker | Which containers have updates available? |
| WAN Monitoring | Speedtest Tracker | Is the Internet connection performing normally? |

This layered approach avoids relying on a single monitoring application.

---

# Monitoring Workflow

When an issue occurs, the monitoring stack provides a structured troubleshooting workflow.

```text
Homepage
   │
   ▼
Service appears unhealthy
   │
   ▼
Uptime Kuma
   │
   ├── Service reachable
   │        │
   │        ▼
   │   Application-level investigation
   │
   └── Service unreachable
            │
            ▼
      Grafana / Prometheus
            │
            ▼
      Check historical metrics
            │
            ▼
       Netdata / Glances
            │
            ▼
      Real-time investigation
```

For example, if Jellyfin becomes unavailable:

1. Homepage provides the first visual indication.
2. Uptime Kuma confirms whether the service endpoint is reachable.
3. Grafana can show whether CPU, memory, disk, or network activity changed before the incident.
4. Netdata or Glances can provide detailed real-time host information.
5. Application logs can then be inspected if infrastructure metrics appear normal.

This reduces troubleshooting from guesswork to a repeatable process.

---

# Monitoring vs Logging

The current environment focuses primarily on **metrics and availability monitoring**.

These are not the same as centralized logging.

Monitoring answers questions such as:

```text
Is the service running?
How much CPU is being used?
How much memory is available?
Is disk usage increasing?
Is network throughput abnormal?
```

Centralized logging answers different questions:

```text
Who authenticated?
Which request failed?
What error occurred?
What security event happened?
Which IP generated the request?
```

At present, logs remain largely distributed across the individual services and hosts.

Centralized log collection is therefore one of the major planned observability improvements.

---

# Security Monitoring Considerations

Monitoring infrastructure is also security-relevant.

Metrics and availability information can help identify abnormal behavior such as:

- Unexpected CPU spikes
- Sudden network traffic increases
- Repeated service failures
- Unexpected container restarts
- Storage exhaustion
- DNS anomalies
- Previously stable services becoming unavailable

However, infrastructure metrics alone do not provide full security detection.

Future security monitoring will require additional telemetry such as:

- Authentication logs
- Firewall logs
- DNS logs
- Reverse proxy logs
- Linux system logs
- Application logs
- Network intrusion detection events

These will complement the existing Prometheus-based monitoring architecture.

More information about the security strategy is documented in:

[Cybersecurity](cybersecurity.md)

---

# Credential Security

Monitoring systems frequently require API credentials.

Credentials used by monitoring services should follow several rules:

- Use dedicated monitoring accounts
- Prefer API tokens over passwords
- Apply least-privilege permissions
- Avoid root credentials
- Do not commit secrets to Git
- Rotate credentials when required
- Store secrets outside publicly tracked configuration files

For example, Proxmox monitoring should use a dedicated read-only monitoring identity rather than the primary administrative account.

Public repository examples therefore use placeholders such as:

```yaml
username: ${PROXMOX_USER}
token: ${PROXMOX_TOKEN}
```

rather than real credentials.

---

# Monitoring Configuration in Git

Reusable monitoring configuration can be included in the repository, but secrets and environment-specific information should be excluded.

A future repository structure can include:

```text
configs/
└── monitoring/
    ├── prometheus/
    │   └── prometheus.example.yml
    ├── grafana/
    │   └── dashboards/
    └── uptime-kuma/
        └── README.md
```

Example configurations should replace:

- IP addresses where appropriate
- API tokens
- Passwords
- Authentication cookies
- Private DNS information

with documented placeholders.

This demonstrates infrastructure configuration without exposing sensitive HomeLab information.

---

# Monitoring Challenges

Implementing the monitoring stack introduced several practical challenges.

## Prometheus Configuration Errors

Prometheus configuration is sensitive to YAML syntax and indentation.

A malformed `prometheus.yml` can prevent the service from starting correctly.

This reinforced the importance of validating configuration changes before restarting infrastructure services.

---

## Exporter Authentication

Infrastructure exporters often require access to management APIs.

Using unrestricted administrative credentials would make setup easier, but would unnecessarily increase risk.

The preferred solution is therefore to create dedicated monitoring identities with only the permissions required to retrieve metrics.

---

## Container Visibility

Monitoring Docker requires distinguishing between:

- Docker host resource usage
- Individual container resource usage
- Application availability

No single metric source provides the complete picture.

This is one reason the monitoring stack combines Prometheus, Uptime Kuma, Netdata, and application-specific integrations.

---

## Monitoring the Monitoring Stack

Monitoring tools themselves can fail.

For example:

```text
Prometheus DOWN
Grafana DOWN
Uptime Kuma DOWN
```

would reduce visibility into the rest of the infrastructure.

This creates an interesting operational challenge:

> Who monitors the monitoring system?

The current design partially addresses this by using overlapping monitoring tools, but external monitoring would provide stronger failure detection.

---

# Lessons Learned

Building the monitoring environment reinforced several infrastructure principles.

### Monitoring should be designed, not added randomly

Installing multiple monitoring applications without assigning clear responsibilities creates unnecessary complexity.

Each tool in the current environment therefore has a defined role.

### Availability and performance are different

A server can have healthy resource metrics while an application is unavailable.

Both service checks and infrastructure metrics are required.

### Historical metrics are extremely valuable

A live dashboard only shows the current state.

Prometheus and Grafana allow problems to be investigated after they occur.

### Least privilege also applies to monitoring

Monitoring services should not receive administrative permissions simply because they require API access.

### Configuration changes should be validated

Monitoring infrastructure is itself production-like infrastructure.

A small YAML error can remove visibility across the environment.

---

# Future Improvements

The monitoring platform will continue evolving as the HomeLab becomes more security-focused.

Planned improvements include:

- Grafana alerting
- Prometheus alert rules
- Alertmanager
- Additional Prometheus exporters
- Pi-hole metrics integration
- Jellyfin metrics expansion
- Home Assistant metrics expansion
- Disk health / SMART monitoring
- External HDD health monitoring
- Backup-job monitoring
- Certificate expiration monitoring
- Centralized logging
- Security-event correlation
- Wazuh integration
- Suricata network monitoring
- Loki for log aggregation
- Automated notifications
- Monitoring configuration as code

A future architecture could evolve toward:

```text
                    Infrastructure
                         │
            ┌────────────┼────────────┐
            │            │            │
         Metrics        Logs       Security
            │            │            │
            ▼            ▼            ▼
       Prometheus      Loki       Wazuh/Suricata
            │            │            │
            └────────────┼────────────┘
                         ▼
                       Grafana
                         │
                         ▼
                 Alerts / Dashboards
```

This would move the HomeLab from basic infrastructure monitoring toward a more complete **observability and security monitoring platform**.

---

# Key Takeaways

The monitoring stack transforms the HomeLab from a collection of self-hosted services into an observable infrastructure environment.

The current implementation demonstrates practical experience with:

- Infrastructure monitoring
- Time-series metrics
- Prometheus
- Grafana
- Linux monitoring
- Docker monitoring
- Service availability monitoring
- Network performance monitoring
- API authentication
- Least-privilege access
- Troubleshooting workflows
- Dashboard design
- Operational observability

Rather than deploying monitoring tools purely for visualization, the objective is to create a monitoring system that can support real troubleshooting, capacity analysis, maintenance, and future security detection.

---

## Related Documentation

- [Architecture](architecture.md)
- [Cybersecurity](cybersecurity.md)
- [Networking](networking.md)
- [Docker & Media Stack](docker-media-stack.md)
- [Backup & Recovery](backup-recovery.md)
- [Lessons Learned](lessons-learned.md)
