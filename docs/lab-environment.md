# Lab Environment

> November 2026 update. The former WebApp container now hosts the Patrixa budgeting application. It is no longer described as an active general-purpose security-testing workload.

## Purpose

The security learning environment is distinct from stable application and infrastructure workloads. The objective is to build practical defensive skills in a controlled setting without exposing operational systems or real personal data to experiments.

![Separate detection learning environment](../diagrams/exported/lab-architecture.svg)

## Confirmed separate Wazuh lab

The project notes describe a working single-node Wazuh SIEM running in a separate virtualized test environment with a monitored Windows endpoint.

| Capability | Evidence status |
| --- | --- |
| Wazuh SIEM installation | Previously confirmed |
| Windows endpoint enrollment | Previously confirmed |
| Authentication-failure event detection | End-to-end detection previously confirmed |
| Additional endpoint telemetry such as Sysmon | Discussed; current state requires verification |
| Integration with the primary Proxmox infrastructure | Planned, not confirmed |
| Dedicated future lab network | Planned |

The evidence is a point-in-time record from the Wazuh project notes, not an assertion of current service health. Current versions, network details, endpoints, credentials and raw logs are private.

## Detection engineering workflow

The defensive learning workflow focuses on collecting approved telemetry, confirming event visibility, reviewing alerts, understanding their context, and documenting what was learned.

For portfolio evidence, use controlled or synthetic test events and screenshots that omit credentials, private system names, real user identities and network addresses. Do not include unrelated production data.

## Separation and recovery principles

- Use dedicated test environments for experiments.
- Keep operational applications and sensitive data outside test workloads.
- Choose virtualization and isolation appropriate to the activity.
- Maintain recoverable test configurations and document changes.
- Separate observed results from planned capabilities.

Patrixa and Averli are application-hosting workloads rather than exercise targets. See [Application hosting](application-hosting.md).

## Future directions

The target lab design includes additional controlled endpoints, independently verified access boundaries, repeatable defensive monitoring scenarios, centralized logging and documented recovery practices.

Potential learning areas include Windows and Linux telemetry, file integrity monitoring, authentication-event analysis, alert investigation, network monitoring and incident-response documentation.

The project will retain only sanitized case studies and demonstrations, with stable infrastructure kept separate from experimentation.

## Related documentation

- [Cybersecurity](cybersecurity.md)
- [Monitoring](monitoring.md)
- [Networking](networking.md)
- [Backup and recovery](backup-recovery.md)
