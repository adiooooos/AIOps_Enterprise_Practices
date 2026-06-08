# AAP Preparation

> **Status**: 20260608 Updated  
> **Series**: AIOps DEMO Center Deploy & Setup Step-by-Step Guide  

## Core Summary

| # | Dimension | Insight |
| --- | --- | --- |
| ① | **Preparation ≠ Install** | This chapter covers host initialization and offline bundle placement; editing `inventory-growth` and running `setup.sh` are in [03-03 AAP Server Deploy](03-03-AAP-Server-Deploy-CN.md). |
| ② | **Operating Identity** | Containerized install uses **`admin`** as the deploy user; some steps (e.g. `useradd`, firewall) require **root**, then switch to `admin`. |
| ③ | **DEMO Environment Baseline** | Static IP, FQDN, `/etc/hosts`, and SSH trust must align with [02-02 Resources Planning](../02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md) and [03-01 Prerequisites](03-01-AAP-Prerequisites-EN.md). |

## Chapter Objective

After [03-01 Prerequisites](03-01-AAP-Prerequisites-EN.md) are met, complete **standardized host preparation** for the AAP Server — offline packages, deploy user, network and hostname, SSH trust, subscription registration, time sync, performance tuning, and installer bundle extraction — ready for containerized installation.

| Item | Description |
| --- | --- |
| **Versions** | RHEL 9.2+ · Ansible Automation Platform **2.6** (Containerized) |
| **Topology** | [Container Growth](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/tested_deployment_models/container-topologies) · single VM |
| **Example Node** | `aap26.example.com` · `10.210.65.24` |
| **Deploy User** | `admin` (example password: `redhat`) |

---

## Executive Summary: Preparation at a Glance

| # | Category | Key Actions | Required |
| --- | --- | --- | --- |
| 1 | **Offline bundle** | Download AAP 2.6 containerized bundle or use shared archive | ✅ |
| 2 | **Deploy user** | Create `admin`, passwordless `sudo`, enable `linger` | ✅ |
| 3 | **Network** | Static IP, gateway, DNS (recommend `nmtui`) | ✅ |
| 4 | **Hostname** | FQDN + `/etc/hosts` (full DEMO stack nodes) | ✅ |
| 5 | **Firewall** | May disable for DEMO/PoC; production uses port policy | Recommended |
| 6 | **SSH trust** | root and admin to localhost and all `/etc/hosts` nodes | ✅ |
| 7 | **Subscription** | Register RHEL 9 via `subscription-manager` | ✅ |
| 8 | **Time sync** | Install and enable `chronyd` | ✅ |
| 9 | **Performance** | Raise `ulimit` `nofile` to 65535 | Recommended |
| 10 | **Dependencies** | `ansible-core`, `wget`, `git`, `rsync`, `vim` | ✅ |
| 11 | **Extract bundle** | Place bundle under `/home/admin` and extract | ✅ |

---

## 1. Offline Package Preparation

### Option A: Baidu Netdisk (AIOps DEMO Pre-built Archive)

| Item | Value |
| --- | --- |
| **Share name** | `01-AIOPS_sws` |
| **Link** | https://pan.baidu.com/s/1cmwuHd5ztjbw4bCg7WwlEQ?pwd=AE86 |
| **Access code** | `AE86` |

> The archive typically includes an AAP 2.6 offline bundle matched to the DEMO environment. Copy it to the AAP Server after download.

### Option B: Red Hat Customer Portal (Online Download)

| Item | Value |
| --- | --- |
| **Product** | Ansible Automation Platform 2.6 · Containerized Setup Bundle |
| **Download page** | https://access.redhat.com/downloads/content/480/ver=2.6/rhel---9/2.6/x86_64/product-software |
| **Example filename** | `ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64.tar.gz` |

> Select the offline installer matching your target RHEL version (x86_64).

---

## 2. Containerized AAP Server Standard Configuration

> **Note**: Run these steps on a fresh RHEL 9 VM. Steps marked **[root]** use root; steps marked **[admin]** run as `admin`. After preparation, subsequent installation runs as `admin`.

### 2.1 Create Deploy User `admin`

As **root**:

```bash
useradd admin
echo redhat | passwd admin --stdin
echo "admin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
loginctl enable-linger admin
```

| Setting | Description |
| --- | --- |
| **Password** | Example `redhat`; use a strong password in production |
| **sudo** | `NOPASSWD: ALL` simplifies unattended install; tighten for production |
| **linger** | Allows `admin` systemd user services when not logged in (required for containerized install) |

### 2.2 Static IP Configuration

Use **`nmtui`** to configure a static IP, **gateway**, and **DNS servers**.

**DEMO example** (`ens33` · `10.210.65.24/24`):

```ini
# /etc/NetworkManager/system-connections/ens33.nmconnection (excerpt)
[ipv4]
address1=10.210.65.24/24,10.210.65.254
dns=10.192.206.245;10.200.0.245;
may-fail=false
method=manual
```

Verify:

```bash
nmcli connection show ens33 | grep ipv4
ip -4 addr show ens33
```

### 2.3 Hostname and `/etc/hosts`

AAP 2.6 requires an **FQDN**. DEMO hostname:

| Item | Value |
| --- | --- |
| **FQDN** | `aap26.example.com` |
| **Short name** | `aap26` |

```bash
sudo hostnamectl set-hostname aap26.example.com
sudo vi /etc/hosts   # Add AAP and full DEMO stack nodes
```

**DEMO `/etc/hosts` example** (aligned with 02-02):

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.210.65.24  aap26.example.com  aap26
10.210.65.17  test-rhel-7.9-1
10.210.65.103 n8n.example.com    n8n
10.210.65.45  win2019
10.210.65.174 prometheus.example.com prometheus
10.210.65.148 Chaos.example.com  Chaos
10.210.65.14  mattermost.example.com mattermost
```

Verify FQDN resolution:

```bash
ping -c1 localhost
ping -c1 $(hostname -f)
hostname --fqdn
# Expected: aap26.example.com
```

### 2.4 Firewall

| Environment | Recommendation |
| --- | --- |
| **DEMO / PoC** | Firewall may be disabled to simplify troubleshooting |
| **Production** | Open Gateway, Controller, Hub, and EDA ports per AAP 2.6 docs |

DEMO disable example (root):

```bash
systemctl stop firewalld
systemctl disable firewalld
```

### 2.5 SSH Mutual Trust

The installer and ongoing operations require **passwordless SSH** from the AAP Server to itself and managed nodes. Configure for both **root** and **admin**.

**Localhost trust (root)**:

```bash
ssh-keygen          # Accept defaults
ssh-copy-id aap26.example.com
ssh-copy-id aap26
ssh-copy-id 10.210.65.24
```

**Localhost trust (admin)**:

```bash
su - admin
ssh-keygen
ssh-copy-id aap26.example.com
ssh-copy-id aap26
ssh-copy-id 10.210.65.24
```

**Extend to other `/etc/hosts` nodes**:

For each target (`test-rhel-7.9-1`, `n8n.example.com`, `prometheus.example.com`, etc.), run `ssh-copy-id <hostname-or-ip>` as both root and admin so install and playbooks can reach all hosts.

Verify:

```bash
ssh aap26.example.com hostname
ssh prometheus.example.com hostname   # example
```

### 2.6 RHEL Subscription Registration

Register the RHEL 9 host with a subscription-enabled account (root or admin + sudo):

```bash
sudo subscription-manager register --username <RHSM_USER> --password <RHSM_PASS>
sudo subscription-manager attach --auto
sudo subscription-manager repos --enable ansible-automation-platform-2.6-for-rhel-9-x86_64-rpms
sudo dnf repolist
```

> Offline environments may use a Manifest or pre-configured local repos. AAP license activation is in [03-04 Configuration](03-04-AAP-Configuration-CN.md).

### 2.7 System Clock (chronyd)

```bash
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
systemctl status chronyd
```

### 2.8 Performance Tuning (`ulimit` nofile)

For large-scale deployments, raise the open-file limit:

```bash
sudo tee /etc/security/limits.d/40-nofile.conf <<'EOF'
* soft nofile 65535
* hard nofile 65535
EOF

cat /etc/security/limits.d/40-nofile.conf
```

> Re-login after changes for limits to take effect in new sessions.

### 2.9 Install Required Packages

```bash
sudo dnf install -y ansible-core wget git-core rsync vim
ansible --version
```

| Package | Purpose |
| --- | --- |
| **ansible-core** | Required to run the containerized installer |
| **wget / git / rsync / vim** | Download, sync, and edit inventory and config |

### 2.10 Copy and Extract Offline Installer Bundle

Copy the bundle to **`/home/admin`**:

```bash
# As admin (or scp/rsync from a jump host)
cd /home/admin
tar xzf ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64.tar.gz
cd ansible-automation-platform-containerized-setup-bundle-2.6-*-x86_64
ls -la
```

The extracted directory should contain `inventory-growth`, `setup.sh`, `README.md`, and other installer files.

---

## Pre-Deployment Checklist

| # | Check | Pass Criteria |
| --- | --- | --- |
| 1 | Deploy user | `admin` exists, passwordless `sudo`, `loginctl show-user admin` shows `Linger=yes` |
| 2 | Static IP | Matches 02-02 plan; gateway and DNS reachable |
| 3 | FQDN | `hostname --fqdn` → `aap26.example.com` |
| 4 | `/etc/hosts` | AAP and related DEMO nodes listed |
| 5 | SSH trust | root and admin: `ssh <host> hostname` without password prompt |
| 6 | RHEL subscription | `subscription-manager status` shows registered |
| 7 | chronyd | `systemctl is-active chronyd` → `active` |
| 8 | nofile | `/etc/security/limits.d/40-nofile.conf` contains 65535 |
| 9 | ansible-core | `ansible --version` succeeds |
| 10 | Offline bundle | Extracted under `/home/admin/...`, `inventory-growth` present |

---

## References

- [03-01 AAP Prerequisites](03-01-AAP-Prerequisites-EN.md)
- [Containerized Installation Guide](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation)
- [02-02 Resources Planning](../02-DEMO_architecture_Planning/02-02-Resources-Planning-EN.md)

> **Next:** [03-03 AAP Server Deploy](03-03-AAP-Server-Deploy-CN.md) — edit `inventory-growth` and run `setup.sh`
