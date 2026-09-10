# Home Assistant

## Overview

Home Assistant is the central smart-home platform of the HomeLab.

It provides a single automation and control layer for smart devices that would otherwise operate through separate vendor ecosystems.

Home Assistant is responsible for:

- Device integration
- Zigbee device management
- Automation logic
- Presence detection
- Smart lighting
- Sensors and switches
- Dashboard visualization
- Remote smart-home access
- Backup and recovery

Rather than deploying Home Assistant as a Docker container, the HomeLab uses a dedicated **Home Assistant OS virtual machine** running on Proxmox VE.

This provides stronger isolation and access to the complete Home Assistant ecosystem, including add-ons, backups, hardware passthrough, and operating-system management.

---

# Architecture

Home Assistant runs as a dedicated virtual machine on Proxmox and acts as the central orchestration layer for smart-home devices, integrations, presence detection, and automations.

![Home Assistant Architecture](../diagrams/exported/home-assistant-architecture.svg)

The architecture uses multiple integration methods:

- **Zigbee** devices communicate through the SONOFF Zigbee coordinator passed through to the Home Assistant VM.
- **Network integrations** connect Wi-Fi and LAN-based smart devices.
- **Bluetooth** observations are extended through ESP32 proxies and processed by Bermuda for presence detection.
- **Automations** combine device states, sensor data, presence information, and conditions to control the home.
- The **3D floor-plan dashboard** provides a spatial interface for interacting with smart-home entities.
- **VPN-based remote access** allows Home Assistant to remain private rather than exposing its administrative interface directly to the Internet.
- **Home Assistant backups** are replicated to Google Drive for off-host recovery.

---

# Proxmox Deployment

Home Assistant runs as a dedicated virtual machine on the Proxmox host.

| Property | Configuration |
|---|---|
| Hypervisor | Proxmox VE |
| VM ID | `100` |
| Operating System | Home Assistant OS |
| Hostname | `haos-17.2` |
| Memory | 4 GB |
| Virtual Disk | 32 GB |
| Network | Home LAN |
| Deployment Type | Full VM |

Using a VM rather than an LXC provides a stronger isolation boundary and simplifies support for Home Assistant OS.

Conceptually:

```text
Lenovo ThinkCentre
        │
        ▼
    Proxmox VE
        │
        ▼
     VM 100
        │
        ▼
Home Assistant OS
        │
        ▼
 Home Assistant Core
```

---

# Why Home Assistant OS?

Home Assistant can be installed using several deployment models.

For this HomeLab, Home Assistant OS was selected because it provides an integrated appliance-like environment.

The installation includes:

```text
Home Assistant Core
Supervisor
Operating System
Add-on support
Backup management
Hardware management
```

This simplifies maintenance compared with manually managing each Home Assistant component.

It also makes the VM relatively self-contained and easy to back up or restore.

---

# Why a Dedicated VM?

Home Assistant controls physical devices and contains important automation state.

Keeping it separate from the Docker media stack provides several advantages.

## Isolation

Problems affecting Docker applications do not directly affect Home Assistant.

```text
Proxmox
   │
   ├── VM 100 → Home Assistant
   │
   └── LXC 102 → Docker
                    │
                    ├── Sonarr
                    ├── Radarr
                    ├── qBittorrent
                    └── Other services
```

## Independent Lifecycle

Home Assistant can be:

- Updated
- Restarted
- Snapshotted
- Backed up
- Restored

without modifying the Docker environment.

## Hardware Access

The VM can receive dedicated USB devices through Proxmox passthrough.

This is particularly important for Zigbee.

---

# Zigbee Architecture

The HomeLab uses a USB Zigbee coordinator connected to the Proxmox host and passed through to Home Assistant.

```text
Zigbee Devices
      │
      │ Zigbee Mesh
      ▼
SONOFF Zigbee Coordinator
      │
      │ USB
      ▼
 Proxmox Host
      │
      │ USB Passthrough
      ▼
 Home Assistant VM
      │
      ▼
 Zigbee Integration
```

The coordinator allows compatible Zigbee devices to communicate locally with Home Assistant rather than depending entirely on vendor cloud platforms.

Examples of Zigbee device categories include:

- Motion sensors
- Contact sensors
- Smart plugs
- Buttons
- Lights
- Environmental sensors

---

# Why Zigbee?

Zigbee provides several useful characteristics for a HomeLab smart-home environment.

### Local Communication

Many Zigbee devices can communicate directly with Home Assistant without requiring Internet connectivity.

### Mesh Networking

Powered Zigbee devices can act as routers and extend the network.

Conceptually:

```text
Coordinator
    │
    ├── Sensor
    │
    ├── Smart Plug
    │       │
    │       └── Sensor
    │
    └── Light
            │
            └── Sensor
```

As the number of powered routing devices increases, the mesh can provide additional paths between devices and the coordinator.

### Vendor Independence

A single Zigbee coordinator can support devices from multiple manufacturers, reducing dependence on separate proprietary hubs.

---

# Wi-Fi and Network Integrations

Not every device in the HomeLab uses Zigbee.

Home Assistant also integrates network-connected devices using:

- Local APIs
- Vendor integrations
- Wi-Fi
- Cloud APIs where required
- MQTT where appropriate

The preferred architecture is:

```text
Local integration
      ↓
Local network communication
      ↓
Cloud integration only when required
```

Local control is preferred because it generally provides:

- Lower latency
- Reduced cloud dependency
- Better resilience during Internet outages
- Greater control over device communication

---

# Example Smart Lighting Architecture

A typical automation combines a physical device, sensor state, and Home Assistant logic.

```text
Motion Sensor
      │
      ▼
Home Assistant
      │
      ├── Check conditions
      │
      ├── Check current state
      │
      └── Apply automation
              │
              ▼
         Smart Light
```

This allows automation behavior to be changed centrally without replacing the underlying devices.

---

# Example: Kitchen Lighting

The kitchen currently contains multiple Home Assistant entities participating in lighting automation.

Examples include:

```text
Main kitchen light
Kitchen LED lighting
Motion / occupancy sensor
```

A simple synchronization automation can ensure that secondary lighting follows the main light.

Conceptually:

```text
Main Light State Changes
          │
          ▼
     Home Assistant
          │
       ┌──┴──┐
       │     │
      ON    OFF
       │     │
       ▼     ▼
 LED ON   LED OFF
```

More advanced logic can incorporate occupancy, time, brightness, or presence.

For example:

```yaml
automation:
  - alias: "Kitchen lighting"
    triggers:
      - trigger: state
        entity_id: binary_sensor.kitchen_occupancy
        to: "on"

    actions:
      - action: light.turn_on
        target:
          entity_id: light.kitchen
```

The example is intentionally generic so that private entity identifiers do not need to be exposed in the public repository.

---

# Automation Design

Home Assistant automations follow an event-driven model.

```text
Trigger
   │
   ▼
Conditions
   │
   ▼
Actions
```

### Trigger

Something happens.

Examples:

```text
Motion detected
Door opened
Device state changed
Time reached
Person arrives home
```

### Condition

Optional logic determines whether the automation should continue.

Examples:

```text
Only after sunset
Only if nobody is sleeping
Only when somebody is home
Only if the light is currently off
```

### Action

Home Assistant performs the required operation.

Examples:

```text
Turn on a light
Send a notification
Activate a scene
Change thermostat
Run a script
```

Separating triggers, conditions, and actions keeps automation logic understandable and maintainable.

---

# Presence Detection

Presence detection is used to make automations aware of whether people are home and, where possible, their approximate location.

The HomeLab has experimented with multiple presence sources, including:

- Home Assistant Companion App
- Mobile-device presence
- Bluetooth-based detection
- Bermuda
- ESP32 Bluetooth proxies
- Wearable devices

The architecture can be represented as:

```text
Phone / Wearable
       │
       │ Bluetooth / Network
       ▼
     ESP32
       │
       ▼
    Bermuda
       │
       ▼
Home Assistant
       │
       ▼
Presence State
       │
       ▼
 Automations
```

This provides a foundation for automations that react to real occupancy rather than relying only on fixed schedules.

---

# ESP32 Bluetooth Proxies

ESP32 devices can extend Bluetooth coverage throughout the home.

Instead of requiring the Home Assistant server itself to receive every Bluetooth signal directly:

```text
Bluetooth Device
       │
       ▼
     ESP32
       │
      Wi-Fi
       │
       ▼
Home Assistant
```

Multiple ESP32 devices can provide distributed Bluetooth observations.

This is particularly useful for room-level presence experiments.

---

# Bermuda Presence Detection

Bermuda can use Bluetooth observations from multiple proxies to estimate which area a Bluetooth device is closest to.

Conceptually:

```text
             Bluetooth Device
                    │
          ┌─────────┼─────────┐
          │         │         │
        ESP32     ESP32     ESP32
       Kitchen    Hall      Bedroom
          │         │         │
          └─────────┼─────────┘
                    │
                 Bermuda
                    │
                    ▼
              Home Assistant
                    │
                    ▼
               Area Presence
```

This can enable automations based on room occupancy rather than simply determining whether somebody is somewhere inside the home.

---

# Dashboard

Home Assistant also acts as the main visual interface for the smart-home environment.

The dashboard provides access to:

- Lighting
- Sensors
- Device states
- Automations
- Presence
- Media
- Infrastructure information
- Smart-home controls

The objective is to make complex automation infrastructure understandable through a simple visual interface.

---

# Interactive 3D Floor Plan

One of the ongoing Home Assistant projects is an interactive 3D representation of the home.

The workflow combines:

```text
Floor Plan
    │
    ▼
3D Model
    │
    ▼
OBJ / MTL Assets
    │
    ▼
Home Assistant
    │
    ▼
3D Floorplan Card
```

The 3D model is stored inside the Home Assistant local assets directory and loaded by the dashboard.

A simplified card configuration looks like:

```yaml
type: custom:floor3d-card
path: /local/Floor_3D/
name: Home
objfile: Home.obj
mtlfile: Home.mtl
```

The model can then be associated with Home Assistant entities.

For example:

```yaml
entities:
  - entity: light.example
    type3d: light
    object_id: "123"
```

This allows smart-home state to be represented spatially rather than only through conventional dashboard cards.

---

# 3D Lighting

Lights can be mapped to objects inside the 3D model.

A conceptual configuration is:

```yaml
- entity: light.example
  type3d: light
  object_id: "123"
  light:
    lumens: 800
    color: "#fff2d0"
    decay: 1
    distance: 500
    shadow: "no"
    vertical_alignment: top
```

This allows the visualization to respond to the state of real Home Assistant entities.

The long-term objective is to create a digital representation where physical devices and their Home Assistant entities correspond to locations inside the house model.

---

# 3D Camera Configuration

The 3D dashboard uses a defined camera position, rotation, and target.

Conceptually:

```yaml
camera_position:
  x: ...
  y: ...
  z: ...

camera_rotate:
  x: ...
  y: ...
  z: ...

camera_target:
  x: ...
  y: ...
  z: ...
```

The exact values depend on the exported 3D model and are therefore environment-specific.

Keeping the camera configuration explicit allows the dashboard to open consistently from the intended perspective.

---

# Remote Access

Home Assistant is not intentionally exposed directly to the public Internet.

Remote administration is performed through encrypted remote-access mechanisms.

The HomeLab uses:

- Tailscale
- WireGuard

Conceptually:

```text
Remote Device
      │
      ▼
Encrypted VPN
      │
      ▼
   Home LAN
      │
      ▼
Home Assistant
```

This keeps the Home Assistant management interface behind the private network boundary.

---

# Security Considerations

Home Assistant is treated as a sensitive workload because compromise could affect physical devices.

The current security approach includes:

- Dedicated VM
- Private network placement
- VPN-based remote access
- No unnecessary public administrative exposure
- Independent backups
- Separation from Docker workloads
- Local device communication where practical

A major future security improvement is network segmentation.

The target architecture is:

```text
Management VLAN
      │
      │ controlled access
      ▼
Home Assistant
      │
      │ required protocols only
      ▼
   IoT VLAN
      │
      ▼
Smart Devices
```

IoT devices should not require unrestricted access to management infrastructure.

More details are documented in:

[Cybersecurity Design](cybersecurity.md)

---

# Backup and Recovery

Home Assistant uses multiple recovery layers.

## Proxmox

Because Home Assistant runs as VM 100, the entire virtual machine can be protected through Proxmox backup mechanisms.

## Home Assistant Backups

Home Assistant also creates application-level backups.

These can contain:

- Configuration
- Automations
- Integrations
- Add-ons
- Dashboard configuration
- Application data

## Off-Host Backup

Home Assistant backups are replicated to Google Drive.

```text
Home Assistant
      │
      ▼
 HA Backup
      │
      ▼
Google Drive
```

This provides an off-host recovery copy independent of the Home Assistant VM.

More details are documented in:

[Backup & Recovery](backup-recovery.md)

---

# Monitoring

Home Assistant can be monitored at several layers.

```text
Proxmox
   │
   └── VM health
         │
         ▼
Home Assistant
         │
         ├── Application state
         ├── Device availability
         └── Automation state
```

Infrastructure monitoring can detect:

- VM CPU usage
- Memory usage
- Storage utilization
- Host availability

Home Assistant itself provides additional visibility into:

- Device availability
- Integration failures
- Automation traces
- Entity history
- System logs

The combination provides both infrastructure-level and application-level visibility.

---

# Troubleshooting Approach

Home Assistant troubleshooting is performed progressively from the infrastructure layer upward.

```text
Proxmox Host
     │
     ▼
VM Running?
     │
     ▼
Network Available?
     │
     ▼
Home Assistant Running?
     │
     ▼
Integration Available?
     │
     ▼
Entity Available?
     │
     ▼
Automation Logic
```

This prevents application troubleshooting from hiding lower-level infrastructure problems.

Useful diagnostic sources include:

```text
Proxmox VM status
Home Assistant system logs
Integration logs
Automation traces
Entity history
Network connectivity
Device availability
```

---

# Design Principles

Several principles guide the Home Assistant implementation.

## Local First

Use local device communication where practical.

## Separate Infrastructure from Automation

Home Assistant handles automation while Proxmox handles virtualization and lifecycle management.

## Avoid Single-Vendor Dependency

Use open standards such as Zigbee and integrations that allow devices from different manufacturers to operate together.

## Automate Behavior, Not Access

Manual control should remain possible even when automation is introduced.

## Keep Automations Understandable

Complex behavior should be broken into manageable automations, scripts, scenes, and helpers rather than creating monolithic logic.

## Back Up Before Major Changes

Significant Home Assistant changes should have a recovery path.

---

# Future Improvements

Planned Home Assistant improvements include:

- Complete the interactive 3D floor-plan dashboard
- Expand room-level presence detection
- Add additional ESP32 Bluetooth proxies where useful
- Improve occupancy-aware automations
- Expand local-only device integrations
- Introduce dedicated IoT network segmentation
- Restrict IoT-to-management communication
- Improve automation documentation
- Add additional environmental sensors
- Improve smart-lighting scenes
- Expand infrastructure visibility inside Home Assistant
- Continue reducing dependence on vendor cloud services

---

# Skills Demonstrated

The Home Assistant implementation demonstrates practical experience with:

- Virtual machine deployment
- Proxmox administration
- USB passthrough
- Zigbee networking
- IoT integration
- Event-driven automation
- YAML configuration
- Presence detection
- ESP32 devices
- Bluetooth proxies
- Smart-home networking
- Dashboard development
- 3D visualization
- Backup design
- VPN remote access
- Troubleshooting across multiple infrastructure layers

---

# Key Takeaways

Home Assistant has evolved from a simple smart-device controller into a major component of the HomeLab.

The project combines:

```text
Virtualization
      +
Networking
      +
IoT
      +
Automation
      +
Presence Detection
      +
Visualization
      +
Security
      +
Backup & Recovery
```

Running Home Assistant as a dedicated Proxmox VM provides a stable foundation while allowing smart-home functionality to evolve independently from the rest of the infrastructure.

The long-term objective is a locally controlled, resilient, observable, and increasingly segmented smart-home platform.

---

# Related Documentation

- [Architecture](architecture.md)
- [Networking](networking.md)
- [Cybersecurity](cybersecurity.md)
- [Backup & Recovery](backup-recovery.md)
- [Monitoring](monitoring.md)
