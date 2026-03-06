# Pi Cluster

This directory contains the Kubernetes manifests and Flux GitOps configuration for my Raspberry Pi home lab cluster (Pi Zoro).

## What's in here

This is a hands-on exploration of running a production-style Kubernetes setup on a Raspberry Pi using K3S. The setup covers:

- **K3S** — lightweight Kubernetes running on a Raspberry Pi
- **FluxCD** — GitOps controller managing the cluster state
- **GitOps workflows** — Git as the single source of truth for all cluster configuration
- **Storage** — persistent storage configuration on the Pi
- **Monitoring stack** — Prometheus and Grafana via Helm
- **Ingress** — Traefik for routing external traffic
- **Security** — secrets management and securing applications
- **Automated image updates** — Renovate and Flux image automation
- **Capstone project** — bringing it all together

## Stack

- K3S
- FluxCD
- Kustomize
- Helm
- Traefik
- Prometheus + Grafana
