# Application Hosting

> November 2026 inventory. Public documentation is limited to confirmed architecture roles, development lessons and recovery practices; live endpoints, data and credentials are omitted.

## Overview

The HomeLab hosts two separate, self-hosted budgeting applications in independent Linux container environments: Patrixa and Averli.

| Application | Role | Confirmed deployment information |
| --- | --- | --- |
| Patrixa | Budgeting application | Dedicated LXC; former WebApp environment repurposed |
| Averli | Budgeting application | Dedicated LXC; PostgreSQL, native authentication, PWA support, scheduled database backup and isolated restore verification |

The applications are maintained independently from the Docker media workload. Their separate deployment environments support independent maintenance and recovery lifecycles.

## Patrixa

Patrixa now occupies the environment formerly documented as a generic development and testing workload. Its role is application hosting, not an active security-testing environment.

The internal application stack, specific data storage design, authentication implementation and backup procedure will be added when verified. They are deliberately not inferred from the architecture of Averli.

## Averli

Averli is a self-hosted budgeting application with a PostgreSQL-backed deployment, native authentication and Progressive Web App support.

Its documented recovery practice includes scheduled database backups and an isolated restore verification procedure. This demonstrates application-level backup and recovery separately from guest-level backup.

## Operational and security considerations

Financial application data should be treated as private. The public portfolio documents application roles and engineering decisions without real transactions, user details, authentication material, database identifiers, deployment addresses or private configuration.

Independent application boundaries support separate update and recovery lifecycles, but do not replace application authentication, authorization, secure maintenance and verified access policy.

## Further work

- Verify and document Patrixa's architecture and recovery workflow.
- Maintain a sanitized recovery checklist for each application.
- Track backup freshness, capacity and restore-test evidence.
- Add reusable examples using synthetic data rather than live configuration.
- Document application maintenance and access-control decisions without publishing operational secrets.

## Related documentation

- [Architecture](architecture.md)
- [Backup and recovery](backup-recovery.md)
- [Networking](networking.md)
- [Cybersecurity](cybersecurity.md)
