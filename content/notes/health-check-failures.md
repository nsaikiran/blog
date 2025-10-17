---
layout: post
title:  "Health check configurations in reliable systems"
description: "Node health check failures can be due to early probe checks"
categories: ["system-design"]
date: 2025-01-30 19:45:31 +0530
author: "Sai Kiran"
comments: false
---

Health checks are ubiquitous and form the foundation of **highly available** (to re-route traffic to healthy nodes) and **reliable** (to ensure healthy nodes are always running) systems.  
From experience, I’ve learned that reviewing and fine-tuning health check probe configurations can often be the key to resolving tricky production issues.

---

## 1. Continuous Container Restarts in Kubernetes

During one production incident, I encountered a situation where several containers in our services were **constantly restarting**, causing in-progress requests to fail. These pods also have high CPU utilization during the restart.

Initially, I was uncertain whether the **high CPU usage** caused the restarts or the **restarts** were causing high CPU usage. After further investigation, it became clear that CPU utilization wasn't the source of the problem.

The pattern revealed that this issue occurred only on **freshly created containers**. Once a container entered this problematic state, it remained that way from creation.  

Kubernetes can create or move containers dynamically, but in certain cases, **new containers** were entering a loop of continuous restarts and high CPU utilization. The root cause turned out to be a **delay in pod creation** — due to underlying infrastructure-related slowness that I didn’t explore further.

This delay caused **liveness and readiness probes** to fail, which in turn triggered pod restarts in an endless cycle.  
By adjusting the probe configurations — particularly the `initialDelaySeconds` — and aligning them with the **actual startup time** of the service, the problem was resolved.

**Reference:** [Configure Liveness, Readiness and Startup Probes | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

---

## 2. EC2 Auto Scaling Group Health Check Failures During OS Upgrade

In another case, we had a **public-facing proxy server** that exposed **static IPs** for a backend service with **dynamic IPs**. This design was necessary for enterprise customers who needed to **allowlist specific IP addresses** in their firewall configurations.

The setup consisted of:
- An **EC2 Auto Scaling Group (ASG)** behind a **Network Load Balancer (NLB)**
- EC2 instances running **Nginx in Docker containers**
- VPC peering between the ASG and the upstream service

The Auto Scaling Group performed regular **health checks** to ensure the configured number of healthy nodes were always available.

When performing a **major OS image upgrade** for these EC2 instances, I had to make several configuration adjustments — including `iptables` and `nftables` changes — to maintain the correct traffic path:

> **Client → NLB → ASG → EC2 → Nginx (Docker) → Upstream Service**

Although I could successfully access the health check endpoint from the **bastion host (jump box)**, the **ASG health checks were failing**.  
Upon deeper inspection, I found that the **health check probes were starting before the instance was fully ready**.

The **cloud-init** script — responsible for setting up the environment and starting services — was taking longer to complete in the new OS image because of additional installation and update tasks.  
By **increasing the delay** before the health checks began, the issue was fully resolved.

**Reference:** [aws_autoscaling_group | Terraform Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/autoscaling_group#health_check_grace_period-1)

---

## Key Takeaways

- Health check misconfigurations can mimic infrastructure or application-level issues.  
- Always verify whether probes are firing **too early** or **too frequently**.  
- Tune parameters like `initialDelaySeconds` or `health_check_grace_period` based on **actual startup time** and **environment initialization latency**.  
- Cloud-init or infrastructure provisioning delays can easily interfere with health checks if not accounted for.

---