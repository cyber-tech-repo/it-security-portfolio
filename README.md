# IT & Cybersecurity Portfolio

This repository contains a curated selection of public-facing technical projects, infrastructure notes, research, and documentation from ongoing personal lab work and self-hosted environments.

Most work published here originates from larger private repositories and long-term learning projects. Public releases are selectively reviewed and sanitized prior to publication to remove sensitive information, environment-specific details, and operationally unsafe configurations where applicable.

The purpose of this repository is to document technical work in a structured, reproducible, and maintainable format while showcasing practical problem-solving, research, and operational workflow.

Additional projects and documentation may be added over time as they are prepared for public release.

---

# Current Project

## Debian 12 + AdGuard Home Deployment

Detailed deployment and configuration walkthrough for installing and configuring AdGuard Home on Debian 12 within a self-hosted environment.

The documentation walks through complete deployment, configuration, hardening, and operational management of a Debian-based AdGuard Home DNS appliance within a self-hosted environment.

Documentation is written as a structured follow-along reference intended for reproducibility and long-term usability.

### System Overview

```mermaid
graph LR
    A[Home Network Devices] --> B[Debian 12 DNS Appliance]
    B --> C[AdGuard Home]
    C --> D[Quad9 DNS-over-HTTPS]
    D --> E[Internet]
```

---

# Notes

Some documentation may intentionally omit:
- credentials
- internal network information
- environment-specific identifiers
- sensitive infrastructure details
- unsafe or unnecessarily exposed configurations

All content is intended for educational, documentation, and portfolio purposes.
