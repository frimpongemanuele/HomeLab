# Backup & Recovery

This document describes the backup, snapshot, and recovery strategy implemented in the HomeLab.

The objective is not only to protect application data, but also to make infrastructure changes and upgrades recoverable.

The environment currently uses multiple recovery mechanisms:

- Proxmox VE snapshots
- Proxmox VM/LXC backups
- Home Assistant backups
- Google Drive off-host backup for Home Assistant
- External HDD storage
- Configuration backups for Docker and infrastructure services
- Recovery procedures tested through real incidents

> [!IMPORTANT]
> Snapshots and backups serve different purposes. Snapshots provide fast rollback before infrastructure changes, while independent backups provide recovery when the original system or storage is unavailable.

---

# Backup Architecture

The current backup strategy follows a layered approach.

![HomeLab Backup and Recovery Architecture](../diagrams/exported/backup-architecture.svg)

The diagram shows both the **current backup implementation** and the **target architecture**.

### Current Architecture

The Proxmox host protects its workloads through multiple mechanisms:

- **Home Assistant** creates application-level backups, with an off-host copy stored in Google Drive.
- **Proxmox VM and LXC backups** are stored on the external 3 TB HDD.
- **Infrastructure configuration** is maintained separately where appropriate, with sanitized configuration and documentation version-controlled through Git.
- The external HDD currently contains both media and Proxmox backups.

This creates an important shared failure domain:

> The 3 TB HDD currently contains both primary media data and infrastructure backups. A physical failure of this disk could therefore affect both datasets.

### Target Architecture

The long-term objective is to separate the storage roles:

```text
Primary Storage
      │
      ▼
Separate Backup Storage
      │
      ▼
Off-site Copy
```

```text
/mnt/media
├── movies
├── tv
└── proxmox-backups
```

This is practical for the current HomeLab, although it creates a shared-storage failure domain. Separating media storage and infrastructure backups is therefore part of the planned improvements.

---

# Proxmox Backup Strategy

Proxmox VE is the central virtualization platform and hosts the majority of the HomeLab workloads.

The environment includes both:

- Virtual Machines
- LXC containers

Important workloads include Home Assistant, Jellyfin, Docker services, Tailscale, Homepage, Pi-hole, and the isolated web application testing environment.

Proxmox provides two complementary recovery mechanisms used in the lab.

## Snapshots

Snapshots are primarily used before potentially disruptive operations.

Examples include:

- operating system upgrades
- package upgrades
- configuration changes
- Docker platform changes
- service migrations
- major Home Assistant changes
- infrastructure experiments

The operational workflow is:

```text
Create snapshot
      │
      ▼
Perform update/change
      │
      ▼
Validate services
      │
   ┌──┴──┐
   │     │
Success Failure
   │     │
   ▼     ▼
Delete  Roll back
snapshot snapshot
```

Snapshots are intentionally treated as **temporary rollback points**, not long-term backups.

---

# Pre-Update Workflow

Before major infrastructure updates, the preferred workflow is:

```text
1. Verify current service health
2. Create snapshot
3. Record relevant configuration
4. Perform update
5. Restart affected workload
6. Validate networking
7. Validate application functionality
8. Review logs
9. Remove snapshot after successful validation
```

This reduces the operational risk associated with upgrades.

A simplified example is:

```bash
pct snapshot <CTID> before-update
```

or for a VM:

```bash
qm snapshot <VMID> before-update
```

If an update introduces a critical problem, the workload can be rolled back to the previous state.

---

# Proxmox Backups

Longer-term recovery uses Proxmox backups rather than snapshots.

The backup target is currently an external HDD mounted on the Proxmox host.

```text
Storage ID: backup-hdd
Mount point: /mnt/media
Backup directory: /mnt/media/proxmox-backups
```

The backup job uses:

```text
Mode: Snapshot
Compression: ZSTD
Schedule: Daily / scheduled maintenance window
Target: External HDD
```

The backup scope includes important VMs and LXC containers.

Unlike snapshots, these backups exist outside the workload's primary virtual disk and can therefore be used for reconstruction after workload corruption or accidental deletion.

---

# Home Assistant Backup Strategy

Home Assistant contains a significant amount of state and configuration, including:

- integrations
- automations
- dashboards
- devices
- entities
- add-ons
- application configuration

Its backup strategy therefore includes an additional off-host layer.

```text
Home Assistant
      │
      ▼
Local HA Backup
      │
      ▼
Google Drive
```

Home Assistant backups are automatically replicated to Google Drive.

This provides protection against scenarios where the Proxmox host or local storage becomes unavailable.

The result is a basic form of geographic/off-site redundancy for one of the most important HomeLab workloads.

---

# Configuration Backups

Infrastructure recovery requires more than virtual machine images.

Important configuration files are also maintained separately where practical.

Examples include:

```text
Docker Compose files
Homepage configuration
Prometheus configuration
Grafana configuration / dashboards
Reverse proxy configuration
Jellyfin configuration
Pi-hole configuration
Proxmox scripts
Monitoring configuration
```

Sanitized versions of selected configurations can be stored in this repository under:

```text
configs/
```

Secrets must never be committed.

Examples of information that must be removed before publishing include:

```text
API keys
VPN credentials
passwords
authentication tokens
private keys
cookie secrets
service credentials
```

Environment variables or `.env` files can be used for sensitive Docker configuration.

A public example can instead be provided as:

```text
.env.example
```

---

# External Storage

The HomeLab uses a 3 TB external HDD for bulk storage.

The drive currently contains:

```text
Media
├── Movies
├── TV
└── Other media

Infrastructure
└── Proxmox backups
```

The filesystem was selected to allow the drive to remain portable between Linux and Windows systems.

The drive is mounted by Proxmox at:

```bash
/mnt/media
```

A critical requirement discovered during operation is that external storage must **not prevent the hypervisor from booting** if the device is disconnected or unavailable.

For this reason, removable storage mounts should use resilient mount options such as:

```text
nofail
```

Conceptually:

```text
UUID=<disk-uuid> /mnt/media exfat defaults,nofail 0 0
```

The real UUID is intentionally omitted from public documentation.

---

# Recovery Case Study: External HDD Mount Failure

One of the most important operational lessons from the HomeLab occurred after configuring the external media disk.

An `/etc/fstab` entry was initially configured in a way that made successful mounting of the external disk part of the normal boot process.

After a restart, the external disk was not available as expected.

Proxmox entered emergency mode with errors similar to:

```text
Timed out waiting for device
Dependency failed for /mnt/media
Dependency failed for local-fs.target
```

As a result:

```text
Proxmox
   │
   ├── Host boot interrupted
   ├── VMs unavailable
   ├── LXC containers unavailable
   └── HomeLab services unavailable
```

## Diagnosis

A local console was connected to the Proxmox host.

The boot process showed that the system was waiting for the external storage device.

The problem was traced to:

```bash
/etc/fstab
```

rather than to Proxmox itself or a network failure.

## Recovery

The system was booted into an emergency shell and the problematic mount entry was disabled.

After correcting the configuration, Proxmox booted normally again.

The final design uses a non-blocking mount strategy:

```text
External HDD unavailable
        │
        ▼
Mount fails
        │
        ▼
nofail permits boot
        │
        ▼
Proxmox continues starting
        │
        ▼
Core infrastructure remains available
```

## Lesson

External or non-critical storage should not become an unnecessary dependency for the hypervisor boot process.

This incident reinforced an important infrastructure principle:

> A failure of secondary storage should degrade the services that depend on that storage, not prevent the entire virtualization host from starting.

---

# Recovery Case Study: Filesystem Damage

A second and significantly more serious incident occurred while modifying the external media disk.

The drive contained an existing media library, but destructive storage commands were used during filesystem and partition configuration.

Commands capable of modifying storage metadata include:

```bash
wipefs
fdisk
gdisk
mkfs
```

These commands can destroy or overwrite:

- partition metadata
- filesystem signatures
- filesystem structures
- directory metadata

Although much of the underlying file data may remain physically present immediately after such operations, the filesystem structures required to locate it can be lost.

---

# Recovery Investigation

Several recovery approaches were evaluated.

## TestDisk

TestDisk was first used to search for recoverable partition structures.

The disk presented inconsistent partition information and the original NTFS filesystem could not be reconstructed reliably.

TestDisk was therefore unable to restore the original directory structure.

## R-Studio

R-Studio detected NTFS remnants and recognized a large portion of the disk.

However, the original directory tree and filenames were no longer sufficiently intact for a clean filesystem-level recovery.

This indicated that the filesystem metadata had suffered significant damage.

## PhotoRec

PhotoRec was then used for file carving.

Unlike filesystem recovery tools, file carving searches the raw disk for known file signatures.

Conceptually:

```text
Damaged Filesystem
       │
       ▼
Raw Disk Sectors
       │
       ▼
File Signature Detection
       │
       ▼
Recovered MKV / MP4 / MOV / etc.
```

Thousands of files were detected, including hundreds of video files.

The trade-off was that file carving does not normally preserve:

- original filenames
- folder hierarchy
- library organization

The recovery process therefore became a combination of data recovery and media-library reconstruction.

---

# Recovery Safety Procedure

Once filesystem damage was discovered, the damaged disk was treated as a source rather than a working disk.

The recovery workflow became:

```text
Damaged 3 TB HDD
       │
       │ Read only / avoid further writes
       ▼
Raw Recovery
       │
       ▼
recup_dir.*
       │
       ▼
Working Recovery Copy
       │
       ├── Movies
       ├── TV
       └── Junk
       │
       ▼
File Verification
       │
       ▼
Metadata Identification
       │
       ▼
Library Reconstruction
```

The raw recovery directories were preserved while recovered files were tested and reorganized separately.

This prevented additional destructive operations from affecting the only remaining recovery source.

---

# Media Reconstruction

Recovered media files often had generic names such as:

```text
f2754560.mp4
f68536320.mkv
f1134856192.mp4
```

Where embedded metadata remained available, it could reveal information such as:

```text
Show.Name.S03E02
Show.Name.S01E01
```

Files were then reconstructed into Jellyfin-compatible structures.

For television:

```text
TV/
└── Show Name/
    └── Season 03/
        └── Show Name S03E02.mkv
```

For movies:

```text
Movies/
└── Movie Name (Year)/
    └── Movie Name (Year).mkv
```

This allowed Jellyfin and the media automation stack to identify the recovered content again.

---

# Storage Safety Lessons

The storage incident resulted in several permanent operating rules.

## Inspect Before Modifying

Before performing any storage operation:

```bash
lsblk
lsblk -f
blkid
fdisk -l
df -h
mount
```

The disk identity, filesystem, mount point, and existing data must be verified first.

## Read-Only First

Unknown disks should initially be mounted read-only.

For example:

```bash
mkdir -p /mnt/test
mount -o ro /dev/sdX1 /mnt/test
```

Only after verifying the contents should write operations be considered.

## Treat Destructive Commands Differently

Commands such as:

```bash
wipefs
mkfs
fdisk
gdisk
```

must be treated as destructive operations.

Before executing them, verify:

```text
Device
Filesystem
Capacity
Serial / model
Existing partitions
Existing data
Backup status
```

## Preserve the Source During Recovery

When recovering damaged storage:

```text
Do not format
Do not repartition
Do not write new files
Do not reuse the disk
```

until the recovery process is complete.

---

# Recovery Scenarios

The current backup strategy is designed to address several failure scenarios.

| Scenario | Recovery Mechanism |
|---|---|
| Failed package update | Proxmox snapshot rollback |
| Broken LXC configuration | Snapshot or Proxmox backup |
| Deleted VM/LXC | Proxmox backup |
| Home Assistant corruption | HA backup |
| Proxmox host failure | Restore workloads from backup |
| Home Assistant host loss | Google Drive backup |
| Docker configuration loss | Compose/config backup |
| External HDD missing during boot | `nofail` mount strategy |
| Storage filesystem damage | Offline recovery procedure |
| Service configuration error | Git/config backup + snapshot |

---

# Backup Limitations

The current design is functional but not yet a complete enterprise-style backup architecture.

The largest limitation is that the external 3 TB HDD currently stores both:

```text
Media data
+
Proxmox backups
```

This means failure of the physical disk could affect both the primary media library and infrastructure backups stored on it.

It therefore does not provide true physical redundancy.

The Home Assistant Google Drive backup is currently the strongest off-host component of the backup architecture.

---

# Future Backup Architecture

The long-term objective is to separate primary storage from backup storage.

```text
                   Proxmox
                      │
          ┌───────────┴───────────┐
          │                       │
    Primary Storage          Backup Storage
          │                       │
      VM / LXC              Proxmox Backups
      Media Data                   │
                                   ▼
                            Separate Device
                                   │
                                   ▼
                         Optional Off-Site Copy
```

Potential future improvements include:

- dedicated backup disk
- separate media and backup storage
- Proxmox Backup Server
- backup retention policies
- automated backup verification
- scheduled restore testing
- encrypted off-site backups
- configuration-as-code backups
- SMART disk monitoring and alerts
- backup health monitoring through Grafana
- notifications for failed backup jobs

A longer-term objective is to move closer to the **3-2-1 backup principle**:

```text
3 copies of important data
2 different storage types
1 off-site copy
```

The current environment partially implements this model but does not yet fully satisfy it.

---

# Disaster Recovery Philosophy

The HomeLab recovery strategy follows a simple principle:

> A backup is only useful if it can actually be restored.

For this reason, the project treats recovery as part of infrastructure engineering rather than simply storing copies of data.

The real incidents encountered during the project resulted in practical experience with:

- Linux boot recovery
- `/etc/fstab` troubleshooting
- filesystem investigation
- partition recovery
- file carving
- data reconstruction
- Proxmox snapshots
- VM/LXC backup strategies
- storage failure domains
- recovery planning

These experiences directly influenced the current architecture and operational procedures.

---

# Key Lessons

The most important lessons from the backup and recovery work are:

1. **Snapshots are not backups.**
2. **Critical backups should not share the same failure domain as primary data.**
3. **External storage should not block hypervisor boot.**
4. **Unknown disks should be inspected and mounted read-only before modification.**
5. **Destructive storage commands require explicit device verification.**
6. **Recovery procedures should be designed before they are needed.**
7. **Configuration backups are as important as application data.**
8. **Off-host backups significantly improve resilience.**
9. **A successful backup job does not guarantee a successful restore.**
10. **Real recovery testing is the best validation of a backup strategy.**

---

# Related Documentation

- [Architecture](architecture.md)
- [Cybersecurity](cybersecurity.md)
- [Monitoring](monitoring.md)
- [Docker Media Stack](docker-media-stack.md)
- [Home Assistant](home-assistant.md)
- [Lessons Learned](lessons-learned.md)
