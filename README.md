# DevOps Lab

My learning journal documenting real-world problems solved, experiments, and technical notes as I build DevOps skills. This is where I document the journey from data analytics to DevOps engineering.

## What's Here

This repository contains troubleshooting notes, configuration guides, and technical documentation from my daily learning. A mix of structured guides, raw notes, and working configurations built up over time.

## Repository Structure

📦 **[docker/](https://github.com/AhsanRahat12/devops-lab/tree/main/docker)**
Docker containerization, troubleshooting, and configuration notes.

🐧 **[linux/](https://github.com/AhsanRahat12/devops-lab/tree/main/linux)**
Linux system administration, security, and display server configuration.

☸️ **[kubernetes/](https://github.com/AhsanRahat12/devops-lab/tree/main/kubernetes)**
Kubernetes orchestration, cluster management, and deployment configurations — including a full home lab setup running on Raspberry Pi with K3S, GitOps workflows with FluxCD, monitoring with Prometheus and Grafana, ingress with Traefik, and secrets management.

## Why This Exists

As I transition from data analytics to DevOps, I'm documenting everything I learn and every problem I solve. This serves as:

- My personal reference documentation
- Proof of hands-on learning
- Evidence of problem-solving methodology
- A resource for others facing similar issues

## Technologies Covered

**Containerization & Orchestration**
- Docker, Docker Desktop
- Kubernetes (K3S, Rancher Desktop)
- Helm

**GitOps & CI/CD**
- FluxCD
- Kustomize
- Renovate (automated image updates)
- GitHub Actions

**Infrastructure & Networking**
- Traefik (ingress)
- Persistent storage on Raspberry Pi
- Flannel CNI

**Monitoring & Observability**
- Prometheus
- Grafana
- AlertManager

**Operating Systems**
- Arch Linux (Hyprland)
- Raspberry Pi OS

**Security & Secrets**
- GPG, SSH, YubiKey
- Kubernetes secrets management
- Credential management with pass

**Tools**
- systemd, git, vim
- k9s, kubectl
- Chezmoi, Mise, DevPod

## Hardware

- Personal workstation running Arch Linux
- Raspberry Pi 5 x2 (Pi Zoro, Pi Luffy) running K3S

## Learning Approach

Each note typically includes:

- The problem I encountered
- The solution that worked
- Why it worked (technical explanation)
- Commands and configuration for future reference

> These are working notes, not polished articles. They're written primarily for my own reference, but shared publicly in case they help others troubleshooting similar issues.
