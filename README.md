# Project Redstone

> A simulated enterprise Linux infrastructure project for **Northwind Outdoor Ltd.**, built alongside my RHCSA studies.

## About

Project Redstone is a long-term practical project designed to develop real-world Linux systems administration skills rather than learning commands in isolation.

I am treating the project as a fictional consultancy engagement: gathering business requirements, designing the infrastructure, implementing it through sprints, documenting the environment, and eventually operating and troubleshooting it.

The environment starts small and grows as new business requirements are introduced.

## What I'm Learning

The project combines RHCSA-focused Linux administration with wider infrastructure and delivery skills:

- Linux system administration and troubleshooting
- Users, permissions, services and package management
- Networking, DNS and time synchronisation
- LVM, filesystems and storage management
- Virtualisation with Proxmox/KVM
- Security, monitoring, backup and recovery
- Infrastructure design (HLD/LLD)
- Requirements traceability and change management
- Jira/Scrum-style infrastructure delivery
- Git and GitHub workflows
- SOPs, runbooks and implementation documentation

## Current Environment

| System | Role | Status |
|---|---|---|
| `HYP01` | Proxmox virtualisation host | ✅ Built & validated |
| `MGMT01` | Administration / bastion host | Planned |
| `CORE01` | Core network services | Planned |
| `OPS01` | Logging & monitoring | Planned |
| `FILE01` | File services | Planned |
| `BACKUP01` | Backup & recovery | Planned |

### HYP01

Sprint 1 established the first Northwind infrastructure platform:

```text
HYP01
├── Proxmox VE 9.2
├── Bootstrap Management Network
└── Northwind Workload Storage
    └── vg_northwind
        └── tp_workload
            └── VM / Container Storage
```

The platform has been validated using a temporary RHEL 10.2 VM, including compute, memory, networking, LVM Thin storage, VM lifecycle operations and resource cleanup.

## Project Status

**Current Phase:** Initial Build / Implementation  
**Sprint 1:** ✅ Implementation Complete  
**Next:** Sprint Review → Retrospective → Sprint 2 Planning

Completed so far:

- Discovery workshops and business requirements
- Requirements Traceability Matrix
- High-Level Design
- Initial design decisions
- Low-Level Design
- Jira implementation backlog and Sprint 1 planning
- HYP01 hardware readiness
- Proxmox installation
- LVM Thin workload storage
- HYP01 platform baseline
- RHEL virtualisation validation

## Repository Structure

```text
project-redstone/
├── project-management/    # Requirements, RTM, planning and governance
├── architecture/          # HLD, LLD, design decisions and diagrams
├── implementation/        # Sprint-by-sprint implementation records
├── operations/            # SOPs, runbooks and troubleshooting
├── inventory/             # Infrastructure and asset records
├── configs/               # Version-controlled configuration
├── scripts/               # Administration and automation
├── incidents/             # Incident and post-incident documentation
├── evidence/              # Sanitised implementation evidence
└── media/                 # Project images and diagrams
```

Implementation work is documented by sprint and Jira work item:

```text
implementation/
└── sprint-01/
    ├── NIM-007-host-readiness/
    ├── NIM-008-proxmox-installation-media/
    ├── NIM-009-proxmox-installation/
    ├── NIM-010-local-storage-architecture/
    ├── NIM-012-platform-configuration/
    └── NIM-014-virtualisation-validation/
```

Each implementation record captures what was changed, why it was changed, how it was validated, problems encountered and lessons learned.

## Project Approach

```text
Business Requirement
        ↓
Architecture & Design
        ↓
Jira Work Item
        ↓
Implementation
        ↓
Validation
        ↓
Documentation
        ↓
Operations
```

The goal is not simply to build a Linux lab, but to practise how infrastructure is **designed, implemented, documented, operated and improved** in a realistic environment.

---

**Current milestone:** Sprint 1 implementation complete — HYP01 built, baselined and validated.
