# LFD459 ? Kubernetes for Application Developers Lab Guide

Welcome to the student lab guide for **LFD459 / CKAD preparation**, delivered by **VEGA TRAINING**.

## How to use this guide

Each chapter corresponds to a section of the LFD459 course. Follow the exercises in order ? later chapters build on earlier ones.

Work on your assigned lab nodes:

| Node | Hostname | Role |
|------|----------|------|
| Control Plane | `controller` | Kubernetes control plane |
| Worker 1 | `worker1` | Worker node |
| Worker 2 | `worker2` | Worker node |

Your username on all nodes is `guru`.

## Lab topology

- Kubernetes version: **1.33.1**
- Container runtime: **containerd**
- CNI: **Calico**
- OS: Ubuntu 22.04

!!! note "Cluster pre-installed"
    The cluster is already installed and ready to use. You do **not** need to run any setup scripts. Start directly with the exercises.

## Quick reference

```bash
# Check cluster status
kubectl get nodes

# Check all pods across namespaces
kubectl get pods --all-namespaces

# Check component logs
journalctl -u kubelet -f

# Lab files location
ls ~/lfd459/
```
