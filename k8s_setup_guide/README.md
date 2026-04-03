# Kubernetes Cluster Setup on AWS EC2 with Docker

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.32-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Runtime-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)

---

## 📌 What is This?

This repository documents the step-by-step process of setting up a **production-style Kubernetes cluster on AWS EC2** using Docker as the container runtime.

The cluster consists of:
- **1 Master Node** — controls and manages the cluster (Control Plane)
- **2 Worker Nodes** — where actual application workloads run

We also deploy a sample **Nginx application** on the cluster to verify everything is working correctly.

---

## 🤔 Why Kubernetes?

In the real world, applications are made up of many containers. Managing them manually becomes very difficult as the number grows. **Kubernetes (K8s)** solves this by:

| Problem | Kubernetes Solution |
|---------|-------------------|
| Running containers manually on each server | Automatically schedules containers across nodes |
| App crashes and needs restart | Auto-restarts failed containers |
| Traffic increases suddenly | Auto-scales containers up or down |
| Deploying new versions without downtime | Rolling updates with zero downtime |
| Containers need to talk to each other | Built-in service discovery and networking |

---

## 🤔 Why Docker as Runtime?

Kubernetes needs a **Container Runtime Interface (CRI)** to run containers. Docker is the most widely known and used container platform, making it a great choice for learning and understanding how Kubernetes manages containers under the hood.

> **Note:** Kubernetes removed native Docker support in v1.24+. We use **cri-dockerd** (a shim/adapter) to bridge Docker and Kubernetes together.

---

## 🏗️ Architecture

```
                        AWS EC2
    ┌──────────────────────────────────────────────┐
    │                                              │
    │   ┌─────────────────────┐                   │
    │   │   Master Node        │  (Control Plane)  │
    │   │   172.31.31.151      │                   │
    │   │  - kube-apiserver    │                   │
    │   │  - kube-scheduler    │                   │
    │   │  - kube-controller   │                   │
    │   │  - etcd              │                   │
    │   └──────────┬──────────┘                   │
    │              │                               │
    │    ┌─────────┴──────────┐                   │
    │    │                    │                    │
    │  ┌─▼──────────────┐  ┌──▼─────────────┐    │
    │  │  Worker Node 1  │  │  Worker Node 2  │    │
    │  │  (kubelet)      │  │  (kubelet)      │    │
    │  │  (kube-proxy)   │  │  (kube-proxy)   │    │
    │  │  (Docker)       │  │  (Docker)       │    │
    │  └─────────────────┘  └─────────────────┘   │
    │                                              │
    └──────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| **AWS EC2** | Virtual machines to host the cluster |
| **Ubuntu 22.04** | Operating system on all nodes |
| **Docker** | Container runtime to run containers |
| **cri-dockerd** | Adapter to connect Docker with Kubernetes |
| **kubeadm** | Tool to bootstrap the Kubernetes cluster |
| **kubelet** | Agent running on each node |
| **kubectl** | CLI tool to manage the cluster |
| **Calico** | Network plugin (CNI) for pod communication |

---

## 📁 Repository Structure

```
├── README.md                     # You are here — project overview
└── kubernetes_cluster_setup.md   # Detailed setup commands with explanations
```

---

## 📋 What We Did — Summary

1. Launched **3 EC2 instances** on AWS (1 master, 2 workers)
2. Installed and configured **Docker** on all nodes
3. Installed **cri-dockerd** to bridge Docker with Kubernetes
4. Installed **kubeadm, kubelet, kubectl** on all nodes
5. Initialized the cluster on the **master node**
6. Installed **Calico CNI** for pod networking
7. Joined both **worker nodes** to the cluster
8. Deployed **Nginx** application to verify the setup
9. Exposed Nginx via **NodePort** and accessed it from browser

---

## 🚀 Quick Start

If you want to replicate this setup, follow the detailed guide:

👉 [kubernetes_cluster_setup.md](./kubernetes_cluster_setup.md)

---

## 📸 Screenshots

> Screenshots of each step are included in the setup guide.

---

## ⚠️ Prerequisites

- AWS account with EC2 access
- 3 EC2 instances (Ubuntu 22.04, t2.medium or higher recommended)
- Security groups configured to allow ports: `6443`, `10250`, `80`, and NodePort range `30000-32767`
- Basic knowledge of Linux terminal

---

## 👨‍💻 Author

> Setup performed and documented as part of learning Kubernetes on AWS.
