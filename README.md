# vaultwarden_infrastructure
## Introduction

This project is a portfolio application aimed at understanding how a full infrastructure is built from scratch around an existing application.

The base application is [Vaultwarden](https://github.com/dani-garcia/vaultwarden) — an alternative server-side implementation of the Bitwarden client API, written in Rust (open source). The goal is to build a complete infrastructure around this application, including cluster deployment, CI/CD, monitoring, logging, and cloud deployment.


## Prerequisites
- **WSL2** with Ubuntu distribution — [Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
- **Docker Engine** — [Install Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- **kubectl + k3s** (deployed on a separate physical machine with Debian, but a VM works just as well)
  - [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)
  - [k3s Quick Start](https://docs.k3s.io/quick-start)
- **git** (on all machines/VMs where project work is done)


## Current State / Roadmap
✅ Stage 1: Local k8s deployment (k3s)

⌛ Stage 2: CI/CD pipeline (GitHub Actions)
- ✅ Fix errors from kube-conform

⬜ Stage 3: Monitoring (Prometheus + Grafana)

⬜ Stage 4: AWS deployment

## Repository Structure

```
├── README.md              — project overview, architecture, quick start
├── docs/                  — project instructions and documentations
└── k8s/                   — k8s manifests
```
