# homelab-infra-proxmox

Notes and docs for my home server setup. Running Proxmox VE on consumer hardware with a handful of VMs and LXC containers for different services. This repo tracks what I've built, what broke, and how I fixed it.

## What's running

| | |
|---|---|
| Hypervisor | Proxmox VE 8.x |
| Containers | Unprivileged LXC for most services |
| VMs | KVM for anything needing full isolation or GPU access |
| Networking | Cloudflare Tunnel — no open ports |
| Storage | ZFS pool on dedicated HDDs, LVM for VM disks |
| Backups | Proxmox snapshots + ZFS snapshots |

## Cloudflare Zero Trust Access

Needed a way to reach the Proxmox UI and other services remotely without port forwarding. Set up `cloudflared` as a systemd service on the host, created a named tunnel, and routed multiple services through it:

- Proxmox web UI
- Immich photo platform
- SSH

DNS for each uses a CNAME to the tunnel UUID rather than a direct A record — learned this the hard way after hitting Cloudflare Error 1000. Proxmox uses a self-signed cert so the tunnel config has `noTLSVerify` on the origin side. Access is gated by Cloudflare Access with an email allowlist before it even hits the Proxmox login page.

## Pi-hole DNS + DHCP

AT&T residential gateways don't let you set custom DNS via DHCP, which makes network-wide ad blocking annoying. Worked around it by disabling DHCP on the gateway entirely and running Pi-hole as the authoritative DHCP server out of an LXC container. All devices get Pi-hole as their DNS without any per-device config.

Had to deal with IPv6 — modern OSes will use IPv6 DNS and skip the filter entirely if you don't account for it. Also worth noting that DoH/DoT and apps with hardcoded DNS servers will bypass Pi-hole regardless, which is expected behavior.

## Immich

Self-hosted photo platform running Immich in Docker Compose inside an unprivileged LXC. Getting Docker to run in an unprivileged container on Proxmox requires:

```
lxc.apparmor.profile: unconfined
features: nesting=1,keyctl=1
```

Hit a bad storage situation where a large photo import filled the LVM thin pool, which cascaded into PostgreSQL failures, container restart loops, and ext4 corruption inside the LXC. Recovered the filesystem with `fsck.ext4`, diagnosed the root cause through container logs and `dmesg`, then redesigned storage to keep media on a separate ZFS dataset away from the system LVM.

UID/GID mapping in unprivileged containers is also worth knowing about — the container's root maps to a high UID on the host, so bind-mounted ZFS datasets need to be owned by the mapped UID (`100000:100000` for most things, `100999:100999` for the PostgreSQL data directory).

## Local LLM

Ollama running in a KVM VM with the GTX 1050 Ti passed through via PCIe. The VM is separate from everything else so the GPU is fully dedicated to inference. Documented in [homelab-llm-qwen](https://github.com/willcoded0/homelab-llm-qwen).

## Storage layout

```
/tank (ZFS)
├── library    — Immich media
├── postgres   — database files
└── backups    — snapshot targets

local-lvm (Proxmox LVM)
├── VM disks
└── LXC root filesystems
```

ZFS for anything where data matters, LVM thin for VM flexibility. Separate datasets so a runaway process in one service can't fill storage for everything else.

## Related

- [homelab-llm-qwen](https://github.com/willcoded0/homelab-llm-qwen)
- [minecraft-dynmap-cloudflare-tunnel](https://github.com/willcoded0/minecraft-dynmap-cloudflare-tunnel)
- [willcoded0.github.io/portfolio](https://willcoded0.github.io/portfolio/)
