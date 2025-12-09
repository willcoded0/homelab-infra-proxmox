# Homelab Infrastructure – Proxmox Virtualization Lab

This repo documents my personal homelab, which I use to learn
virtualization, Linux administration, hardware troubleshooting, and
GPU-accelerated workloads.

I rebuilt an older desktop into a Proxmox VE server and use it to host:
- Linux virtual machines (VMs)
- LXC containers
- Game servers (e.g., Minecraft)
- Web tools like Dynmap
- Local AI / LLM inference workloads

---

## 🧱 Hardware Overview

- Repurposed desktop converted into a 24/7 homelab server
- Replaced the **motherboard** and **GPU** to fix instability and extend the life of the system
- Multiple drives for:
  - VM storage
  - Backups
  - Game servers and tools

---

## 🖥️ Proxmox VE Setup

- Proxmox VE as the hypervisor
- Mix of VMs and LXC containers for different services
- Basic networking with Linux bridges and static IPs for key services
- GPU assigned to a VM / container for AI inference testing

---

## 🌐 External Access

- Some services (like Minecraft and Dynmap) are exposed over the internet
  using **Cloudflare Tunnel**, instead of opening ports directly on my router.

This homelab acts as a small-scale lab where I treat my home hardware like
a mini data center to practice real-world systems engineering skills.

