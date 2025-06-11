# 🏠 K3s Distributed Homelab Cluster

This repository houses the configuration, setup guides, and operational details for my personal multi-node Kubernetes homelab cluster. The goal is to build a resilient, geographically distributed, and highly available platform for hosting various services, from web applications to media streaming.

## 🚀 Overview

My cluster leverages K3s, a lightweight Kubernetes distribution, across diverse hardware and locations, including cloud virtual machines, local bare-metal servers, and even edge devices. Secure inter-node communication is established using WireGuard VPNs, and Rancher provides a centralized management interface.

This repository is structured to provide clear instructions for replicating and expanding this setup, as well as managing the lifecycle of applications and infrastructure within the cluster.

## 🗺️ Cluster Network Architecture

Here's a visual representation of the current and planned network topology for the K3s cluster:
![Kuberneties Network](https://github.com/user-attachments/assets/cfc43137-55d7-4e02-889d-5ba398560599)



## 📖 Getting Started: Initial Cluster Configuration

For detailed, step-by-step instructions on setting up the initial K3s master and worker nodes, including WireGuard configuration, Rancher integration, and SSL certificate setup, please refer to the dedicated guide:

* [**Initial Cluster Configuration Guide**](./docs/initial_config.md)

## 📦 Planned Future Documentation

* **Persistent Storage Integration:** Guides on setting up and managing persistent storage solutions (e.g., local PVs for Jellyfin, distributed storage).

* **Application Deployment Examples:** Helm charts and manifests for deploying common homelab applications.

* **Monitoring and Logging:** Setup guides for Prometheus, Grafana, and Loki.

* **Edge Node Integration:** Specific instructions for incorporating low-resource edge devices into the cluster.

---

## 🛠️ Technologies Used

* **Kubernetes:** K3s

* **Container Runtime:** containerd (default with K3s), Docker (if explicitly configured)

* **Ingress Controller:** Traefik

* **Cluster Management:** Rancher

* **VPN:** WireGuard

* **CNI:** Flannel (VXLAN)

* **Certificate Management:** Cert-Manager

* **Cloud Provider:** Oracle Cloud Infrastructure (OCI)

* **Virtualization:** Proxmox VE

* **Operating System:** Ubuntu 24.04 LTS

## 🤝 Contributing

This is primarily a personal homelab project, but suggestions for improvements or best practices are always welcome!

---
