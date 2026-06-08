# AAP Prerequisites

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Core Insight

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Topology** | The AIOps DEMO uses **AAP 2.6 Container Growth** on a **single node** — Gateway, Controller, Hub, EDA, and Database co-located. |
| ② | **Two Spec Tiers** | **Official minimum** satisfies the installer; **DEMO recommended** (see 02-02) reserves headroom for EDA, MCP, and concurrent jobs. |
| ③ | **Prerequisites ≠ Preparation** | This chapter freezes *whether you can install*; host init, offline bundles, SSH trust, etc. are in [03-02 Preparation](03-02-AAP-Preparation-EN.md). |

## Chapter Objective

Confirm that the AAP Server in the AIOps Solution environment **meets containerized installation prerequisites** — hardware, OS, permissions, subscription, time sync, and network baseline — before preparation and install.

| Item | Description |
| --- | --- |
| **Versions** | RHEL 9.2+ · Ansible Automation Platform **2.6** (Containerized) |
| **Topology** | [Container Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies) · single VM |
| **Example Node** | `aap26.example.com` · `10.210.65.24` (see [02-02 Resources Planning](../02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md)) |

---

## Executive Summary: Prerequisites at a Glance

| Category | Requirement | Required |
| --- | --- | --- |
| **OS** | RHEL 9.2+ (DEMO baseline: 9.6) | ✅ |
| **Hardware** | Official minimum or DEMO recommended (tables below) | ✅ |
| **Permissions** | `sudo` / root; ability to drop privileges to AWX, PostgreSQL, EDA, Pulp service users | ✅ |
| **Time sync** | NTP / chronyd on all nodes | ✅ |
| **Subscription** | RHEL subscription + AAP subscription or Manifest | ✅ |
| **Network** | Resolvable FQDN; Internet recommended (images / subscription) | Recommended ✅ |
| **Container runtime** | Podman (default on RHEL 9) available | ✅ |

---

## Hardware & Storage Requirements

### Official Minimum (Container Growth · Single VM)

> Source: AAP 2.6 containerized installation docs · Growth topology tested configuration

| VMs | Resource | Minimum | Co-located Container Roles |
| --- | --- | --- | --- |
| **1** | **RAM** | 16 GB | automationgateway |
| | **CPU** | 4 vCPU | automationcontroller |
| | **Local disk** | 200 GB (**`/home` at least 100 GB**) | automationhub |
| | **Disk IOPS** | 3000 | automationeda |
| | | | database |

### AIOps DEMO Recommended

> Above official minimum — headroom for concurrent jobs, EDA Event Streams, and future MCP integration. See [02-02 Resources Planning](../02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md).

| Resource | DEMO Recommended | Notes |
| --- | --- | --- |
| **vCPU** | 8 | Multi-container single host + concurrent playbooks |
| **RAM** | 20 GB | Controller + EDA + Hub peaks |
| **Disk** | 500 GB | Images, database, project sync, logs |
| **Disk IOPS** | ≥ 3000 | Meets official baseline |

---

## Operating System & Subscription

### Operating System

| Requirement | Description |
| --- | --- |
| **Version** | Red Hat Enterprise Linux **9.2 or later** |
| **DEMO baseline** | RHEL **9.6** (aligned with 02-02 environment) |
| **Architecture** | x86_64 |
| **Registration** | Register via `subscription-manager` and attach repos (03-02) |

### AAP License

| Scenario | Requirement |
| --- | --- |
| **Online** | Valid AAP subscription; activate with subscription credentials after install |
| **Offline / disconnected** | Import enterprise **AAP Manifest** (see [03-04 Configuration](03-04-AAP-Configuration-EN.md)) |

---

## Permissions & Account Model

The containerized AAP installer requires the following (install run as `admin` or equivalent):

| # | Prerequisite | Description |
| --- | --- | --- |
| 1 | **Root access** | Obtain root via `sudo` or privilege escalation |
| 2 | **Privilege drop** | Installer can drop from root to service users: **AWX**, **PostgreSQL**, **Event-Driven Ansible**, **Pulp**, etc. |
| 3 | **Deploy user** | Dedicated `admin` user recommended (`NOPASSWD sudo` + `loginctl enable-linger`) — see 03-02 |
| 4 | **SSH** | SSH trust from install node to local FQDN / IP (Ansible local connection) |

> **Security note:** Avoid long-term root-only ops in production; DEMO/PoC may simplify per 03-02; tighten sudo and SSH before go-live.

---

## Time Synchronization

| Requirement | Description |
| --- | --- |
| **Service** | **chronyd** on all nodes (AAP and future managed hosts) |
| **Accuracy** | Inter-node drift < 1s recommended (EDA alert timestamps, TLS validation) |
| **Verify** | `systemctl status chronyd` · `chronyc tracking` |

---

## Network & Hostname

| # | Prerequisite | Description |
| --- | --- | --- |
| 1 | **FQDN** | Hostname must be FQDN, e.g. `aap26.example.com` (not short name `aap26`) |
| 2 | **Forward / reverse DNS** | Resolvable via `/etc/hosts` or internal DNS; `ping $(hostname -f)` succeeds |
| 3 | **Static IP** | Fixed management IP (example `10.210.65.24/24`) |
| 4 | **Outbound Internet** | **Recommended:** subscription, container image pull, collection download |
| 5 | **Inbound ports** | Post-deploy Web UI **443/tcp** (see 02-02 port matrix) |
| 6 | **Cluster connectivity** | SSH / HTTPS reachable to Prometheus, Chaos, n8n, Mattermost, etc. |

---

## Software & Runtime Dependencies

| Component | Requirement |
| --- | --- |
| **Podman** | Included on RHEL 9; containerized AAP runtime |
| **ansible-core** | Required for installer execution (pre-installed in 03-02) |
| **Toolchain** | `wget` · `git` · `rsync` · `vim` (03-02) |
| **Installer bundle** | `ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64.tar.gz` (download/extract in 03-02) |

**Official documentation:**

- [Containerized Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation)
- [Container Topologies](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies)

---

## Pre-Deployment Checklist

| # | Check | Pass Criteria |
| --- | --- | --- |
| 1 | Hardware | ≥ official minimum; DEMO recommends ≥ 8 vCPU / 20 GB / 500 GB |
| 2 | `/home` space | ≥ 100 GB (container images and data) |
| 3 | Disk IOPS | ≥ 3000 |
| 4 | RHEL version | ≥ 9.2; subscribed and registered |
| 5 | AAP license | Subscription or Manifest ready |
| 6 | FQDN | `hostname --fqdn` returns full domain |
| 7 | chronyd | Service active & enabled |
| 8 | root / sudo | Deploy user can escalate |
| 9 | Internet / offline bundle | Online image pull, or offline bundle in place |
| 10 | Architecture alignment | IP / hostname matches [02-02](../02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md) |

> **Next:** [03-02 AAP Preparation](03-02-AAP-Preparation-EN.md) — user setup, IP/hosts, firewall, SSH trust, offline bundle extraction
