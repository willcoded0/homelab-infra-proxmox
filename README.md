# Proxmox Homelab Infrastructure

Production-grade virtualization infrastructure running on Proxmox VE, hosting containerized services, network infrastructure, and self-hosted applications.

This repository documents the architecture, configuration, and operational practices for a multi-service homelab environment designed with reliability, security, and scalability in mind.

---

## Table of Contents

- [Infrastructure Overview](#infrastructure-overview)
- [Projects](#projects)
  - [Network-Wide DNS Filtering (Pi-hole)](#network-wide-dns-filtering-with-pi-hole)
  - [Self-Hosted Photo Platform (Immich)](#self-hosted-photo-platform-with-immich)
  - [Local LLM Deployment (Ollama)](#local-llm-deployment)
- [Storage Architecture](#storage-architecture)
- [Skills Demonstrated](#skills-demonstrated)

---

## Infrastructure Overview

| Component | Implementation |
|-----------|----------------|
| Hypervisor | Proxmox VE 8.x |
| Virtualization | LXC containers (unprivileged), KVM VMs |
| Storage | ZFS pools on dedicated HDDs, LVM thin provisioning |
| Networking | Cloudflare Tunnel (zero-trust), internal bridges |
| GPU | PCIe passthrough for ML workloads |
| Backup | Automated Proxmox snapshots and ZFS snapshots |

```
Internet
    │
    ▼
Cloudflare Tunnel (Zero Trust)
    │
    ▼
┌─────────────────────────────────────────────┐
│            Proxmox VE Host                  │
├─────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐  │
│  │ Pi-hole │  │ Immich  │  │ Ollama LLM  │  │
│  │  (LXC)  │  │  (LXC)  │  │    (VM)     │  │
│  └─────────┘  └─────────┘  └─────────────┘  │
│                    │              │         │
│              ┌─────┴─────┐   GPU Passthrough│
│              │ ZFS Pool  │                  │
│              │  /tank    │                  │
│              └───────────┘                  │
└─────────────────────────────────────────────┘
```

---

## Projects

### Network-Wide DNS Filtering with Pi-hole

End-to-end deployment of network-wide DNS filtering using Pi-hole in a Proxmox LXC container, integrated with an ISP-locked residential gateway.

#### Problem Statement

AT&T residential gateways do not permit custom DNS server configuration via DHCP, preventing standard network-wide ad blocking implementations.

#### Architecture

| Layer | Component |
|-------|-----------|
| Hypervisor | Proxmox VE |
| Container | Unprivileged LXC |
| DNS Filter | Pi-hole |
| Upstream DNS | Cloudflare (DNSSEC enabled) |
| DHCP | Pi-hole (authoritative) |

```
Clients ──▶ Pi-hole (DHCP + DNS) ──▶ Cloudflare DNS ──▶ Internet
        ▲
        └── AT&T Gateway (Routing only, DHCP disabled)
```

#### Technical Challenges Resolved

**1. Router DNS Lockdown**

AT&T gateway DHCP settings are read-only. Solution:
- Disabled DHCP on the gateway
- Configured Pi-hole as authoritative DHCP server
- All clients automatically receive Pi-hole as DNS without per-device configuration

**2. IPv6 DNS Bypass**

Modern operating systems prioritize IPv6 DNS, silently bypassing IPv4-only filtering.
- Identified IPv6 DNS advertisements via Router Advertisements
- Disabled IPv6 DNS paths where necessary
- Verified enforcement using `pihole -t` query tracing

**3. Encrypted DNS and Application Bypass**

Mobile applications using DoH/DoT or first-party endpoints cannot be filtered at DNS level.
- Documented expected limitations of DNS-level filtering
- Differentiated between bypass and misconfiguration during troubleshooting

#### Verification Methods

- Client-side DNS inspection (`ipconfig /all`, `resolvectl`)
- Forced resolution tests against known blocked domains
- Live query tracing with `pihole -t`
- Per-client traffic auditing via Pi-hole dashboard

#### Results

- Network-wide ad and tracker blocking without client configuration
- DNSSEC-validated upstream resolution
- Centralized DHCP management and device visibility
- Stable, low-latency DNS resolution

---

### Self-Hosted Photo Platform with Immich

Deployed a self-hosted photo management platform replacing cloud photo services, running Immich inside Docker Compose within an unprivileged Proxmox LXC container.

#### Architecture

| Component | Implementation |
|-----------|----------------|
| Container | Unprivileged LXC on Proxmox VE |
| Application | Immich (server, microservices, ML) |
| Orchestration | Docker Compose |
| Database | PostgreSQL |
| Cache | Redis |
| Storage | ZFS pool with bind mounts |

```
┌─────────────────────────────────────────┐
│           Proxmox VE Host               │
│  ┌───────────────────────────────────┐  │
│  │     LXC Container (Unprivileged)  │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │      Docker Compose         │  │  │
│  │  │  ┌───────┐ ┌────┐ ┌─────┐   │  │  │
│  │  │  │Immich │ │ PG │ │Redis│   │  │  │
│  │  │  └───────┘ └────┘ └─────┘   │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
│                  │                      │
│    ┌─────────────┴─────────────┐        │
│    │      ZFS Pool (/tank)     │        │
│    │  /tank/library (media)    │        │
│    │  /tank/postgres (db)      │        │
│    └───────────────────────────┘        │
└─────────────────────────────────────────┘
```

#### Technical Challenges Resolved

**1. AppArmor Blocking Docker Inside LXC**

Docker daemon failed to start due to AppArmor restrictions in unprivileged containers.

Solution:
```
lxc.apparmor.profile: unconfined
features: nesting=1,keyctl=1
```

**2. Storage Exhaustion and Filesystem Corruption**

Large-scale photo uploads exhausted Proxmox `local-lvm` thin pool, causing:
- PostgreSQL write failures
- Container restart loops
- ext4 filesystem corruption inside LXC

Resolution:
- Diagnosed root cause via container logs and `dmesg`
- Performed filesystem recovery using `fsck.ext4`
- Redesigned storage architecture to separate OS, application, and media storage

**3. ZFS Migration for Scalability**

Provisioned dedicated ZFS pool on raw HDDs:
- Configured proper block alignment and compression
- Created separate datasets for media and database
- Mounted into LXC via bind mounts
- Eliminated thin-pool exhaustion risk

**4. UID/GID Mapping in Unprivileged Containers**

Unprivileged LXC containers remap UIDs, causing permission failures.

Resolution:
- Media storage: `chown 100000:100000`
- PostgreSQL data: `chown 100999:100999`
- Verified write access inside containers

**5. Database Integrity Issues**

Encountered after storage recovery:
- Duplicate asset checksum conflicts
- Missing media files referenced in database (`ENOENT` errors)
- Corrupted media headers (HEIC, PNG, MP4)

Resolution:
- Used `psql` inside container to inspect asset metadata
- Traced file paths from database to filesystem
- Performed selective cleanup and re-initialization

#### Operational Practices

- Full teardown and rebuild of Docker Compose stack
- Mount verification using `docker inspect`
- Differentiated Docker volumes vs bind mounts
- Network accessibility testing (LAN, ports, service health)

#### Results

- Stable photo platform handling large uploads and ML workloads
- Fault-tolerant storage architecture
- Complete data ownership without third-party dependencies

---

### Local LLM Deployment

Self-hosted large language model deployment using Ollama with GPU acceleration, exposed as a LAN-accessible API endpoint.

See: [homelab-llm-qwen](https://github.com/willcoded0/homelab-llm-qwen)

#### Implementation

| Component | Configuration |
|-----------|---------------|
| Model | Qwen |
| Runtime | Ollama |
| Virtualization | KVM VM with GPU passthrough |
| Access | REST API on local network |

#### Technical Details

- Configured PCIe passthrough for GPU acceleration
- Deployed Ollama runtime with model persistence
- Exposed API endpoint with access controls
- Integrated with other homelab services

---

## Storage Architecture

```
/tank (ZFS Pool)
├── /tank/library      # Immich media storage
├── /tank/postgres     # PostgreSQL data
└── /tank/backups      # Automated backup targets

local-lvm (Proxmox)
├── VM disks
└── LXC root filesystems
```

Design decisions:
- ZFS for data integrity (checksumming, compression)
- Separate datasets for different failure domains
- LVM thin provisioning for VM flexibility
- Bind mounts for container access to ZFS datasets

---

## Skills Demonstrated

| Category | Skills |
|----------|--------|
| Virtualization | Proxmox VE, LXC containers, KVM, GPU passthrough |
| Linux Administration | systemd, filesystem management, permission models, process debugging |
| Networking | DNS architecture, DHCP, IPv4/IPv6, Cloudflare Zero Trust |
| Storage | ZFS pool management, LVM, bind mounts, data recovery |
| Containers | Docker, Docker Compose, container orchestration |
| Databases | PostgreSQL administration, data integrity, backup/recovery |
| Security | Unprivileged containers, AppArmor, network segmentation |
| Troubleshooting | Log analysis, filesystem repair, performance diagnosis |

---

## Related Repositories

- [minecraft-dynmap-cloudflare-tunnel](https://github.com/willcoded0/minecraft-dynmap-cloudflare-tunnel) - Game server with secure external access
- [homelab-llm-qwen](https://github.com/willcoded0/homelab-llm-qwen) - Local LLM deployment documentation

---

## Contact

- Portfolio: [willcoded0.github.io/portfolio](https://willcoded0.github.io/portfolio/)
- LinkedIn: [william-hall-7091572a0](https://www.linkedin.com/in/william-hall-7091572a0/)
- GitHub: [willcoded0](https://github.com/willcoded0)
