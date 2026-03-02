# 🔴 OpenShift 4.18 — Assisted Installer Deployment Guide

> **Cluster:** `shivang` · **Version:** `4.18.33` · **Platform:** Bare-Metal / VMware vSphere
> **Author:** Shivang Bhardwaj 

![OCP Version](https://img.shields.io/badge/OCP-4.18.33-red?style=flat-square&logo=redhat)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.31.14-326CE5?style=flat-square&logo=kubernetes)
![Status](https://img.shields.io/badge/Install-Success%20✓-brightgreen?style=flat-square)
![Nodes](https://img.shields.io/badge/Nodes-3%20Masters%20%2B%203%20Workers-blue?style=flat-square)
![Duration](https://img.shields.io/badge/Duration-~3h%2048m-orange?style=flat-square)

---

## 📋 Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites & Hardware](#2-prerequisites--hardware)
3. [Accessing the Assisted Installer](#3-accessing-the-assisted-installer)
4. [Cluster Configuration](#4-cluster-configuration)
5. [Generating & Booting the Discovery ISO](#5-generating--booting-the-discovery-iso)
6. [Host Discovery](#6-host-discovery)
7. [Pre-Installation Validation](#7-pre-installation-validation)
8. [Starting the Installation](#8-starting-the-installation)
9. [Monitoring Progress](#9-monitoring-progress)
10. [Issues Encountered & Resolutions](#10-issues-encountered--resolutions)
11. [Post-Installation Verification](#11-post-installation-verification)
12. [Adding a Failed Worker Node](#12-adding-a-failed-worker-node)
13. [Deployment Timeline](#13-deployment-timeline)
14. [Quick Reference Commands](#14-quick-reference-commands)
15. [Key Learnings & Tips](#15-key-learnings--tips)

---

## 1. Overview

This is a complete step-by-step reference for deploying a multi-node OpenShift Container Platform (OCP) 4.18 cluster using the **Red Hat Assisted Installer**. It covers everything from prerequisites to post-install verification, including all real issues encountered and their resolutions.

> **What is the Assisted Installer?**
> A web-based tool at [console.redhat.com](https://console.redhat.com) that provides smart defaults, pre-flight validations, and a guided installation flow — no manual installer download needed.

### Deployment Summary

| Item | Detail |
|------|--------|
| Cluster Name | `shivang` |
| Base Domain | `lab.kuberox.net` |
| OCP Version | `4.18.33` |
| Kubernetes Version | `v1.31.14` |
| Platform | Bare-Metal (VMware vSphere VMs) |
| Control Plane Nodes | 3 (`ocp-master1`, `master2`, `master3`) |
| Worker Nodes | 3 (`ocp-worker1`, `worker2`, `worker3`) |
| Install Duration | ~3 hours 48 minutes (15:16 → 19:05) |
| Install Result | ✅ Completed Successfully |

---

## 2. Prerequisites & Hardware

### 2.1 Hardware Requirements

| Node | Role | OS | RAM | Storage | CPUs |
|------|------|----|-----|---------|------|
| ocp-master1 | Control Plane | RHCOS | 16 GB | 200 GB | 8 |
| ocp-master2 | Control Plane | RHCOS | 16 GB | 200 GB | 8 |
| ocp-master3 | Control Plane | RHCOS | 16 GB | 200 GB | 8 |
| Bastion | Management | RHEL | 16 GB | 100 GB | 8 |
| ocp-worker1 | Worker | RHCOS | 32 GB | 200 GB | 16 |
| ocp-worker2 | Worker | RHCOS | 32 GB | 200 GB | 16 |
| ocp-worker3 | Worker | RHCOS | 32 GB | 215 GB | 16 |

### 2.2 Pre-Installation Checklist

- [ ] Clean all nodes — remove existing virtual disks or old RAID configurations
- [ ] Reserve and document **static IPs** for all nodes (masters, workers, bastion)
- [ ] Document **MAC addresses** for all nodes — needed for NIC assignment in the installer
- [ ] Ensure Bastion is up, reachable over SSH, and using public key authentication
- [ ] Verify **DNS** resolves all node hostnames, API VIP, and Ingress VIP (forward + reverse)
- [ ] Plan **API VIP** and **Ingress VIP** — must be free IPs in the same subnet as nodes
- [ ] Have your Red Hat account credentials and **pull secret** ready

### 2.3 DNS Records Required

> ⚠️ **Verify DNS from bastion before starting:**
> ```bash
> dig api.shivang.lab.kuberox.net
> dig +short ocp-master1.shivang.lab.kuberox.net
> ```

| DNS Record | Points To | Purpose |
|------------|-----------|---------|
| `api.<cluster>.<domain>` | API VIP | External API access |
| `api-int.<cluster>.<domain>` | API VIP | Internal cluster API |
| `*.apps.<cluster>.<domain>` | Ingress VIP | All application routes |
| `ocp-master[1-3].<cluster>.<domain>` | Master node IPs | Node hostname resolution |
| `ocp-worker[1-3].<cluster>.<domain>` | Worker node IPs | Node hostname resolution |

---

## 3. Accessing the Assisted Installer

1. Open a browser and go to **[https://console.redhat.com](https://console.redhat.com)**
2. Log in with your Red Hat account
3. Click **Red Hat Hybrid Cloud Console** in the top navigation
4. Navigate to: **OpenShift → Resources → Assisted Installer**
5. Click **"Create Cluster"** to begin a new deployment

---

## 4. Cluster Configuration

### 4.1 Basic Details

| Setting | Value Used |
|---------|-----------|
| Cluster Name | `shivang` |
| Base Domain | `lab.kuberox.net` |
| OpenShift Version | `4.18.33` |
| CPU Architecture | `x86_64` |
| Network Type | `OVNKubernetes` |

### 4.2 Networking Configuration

| Network Setting | Value | Notes |
|----------------|-------|-------|
| Cluster Network CIDR | `10.128.0.0/14` | Pod network |
| Host Prefix | `/23` | 512 IPs per node |
| Service Network CIDR | `172.30.0.0/16` | Service ClusterIPs |
| Machine Network CIDR | Match your node subnet | Where node IPs live |
| API VIP | Free IP in node subnet | Must not conflict with any host |
| Ingress VIP | Free IP in node subnet | Separate IP from API VIP |

> ⚠️ **Important:** API VIP and Ingress VIP must be in the **same subnet** as your node IPs and must **not conflict** with any existing host.

### 4.3 Static IP / MAC Configuration

- For each node: set static IP, subnet mask, gateway, DNS server, and MAC address
- The Assisted Installer uses **nmstate YAML format** for static IP configuration
- Associate each MAC address to the correct node in the Host Detail view

### 4.4 SSH Public Key

Paste the bastion's SSH public key so you can SSH to RHCOS nodes after install.

Generate on bastion if not present:

```bash
ssh-keygen -t rsa -b 4096
```

---

## 5. Generating & Booting the Discovery ISO

After completing cluster configuration, the installer generates a Discovery ISO. Each node boots from this ISO, installs RHCOS, and an agent communicates back to the Assisted Installer.

1. Click **"Generate Discovery ISO"** in the Assisted Installer UI
2. Choose **Minimal ISO** (recommended — smaller download, fetches remaining data from the network)
3. Download the ISO to your bastion or local machine
4. In VMware vSphere: attach the ISO to each VM via the CD/DVD drive (use datastore ISO or URL)
5. Set boot order so **CD-ROM boots first**, then power on each node
6. Nodes will appear in the **Host Discovery** section of the UI within a few minutes
7. Once a node is discovered, **remove the ISO** from the CD/DVD drive so it boots from disk later

> ⚠️ **If a node does not appear after 5 minutes:**
> - Check the VM console for boot errors
> - Verify the ISO is attached and boot order is correct
> - Confirm network connectivity

---

## 6. Host Discovery

Once all nodes have booted from the ISO, they appear in the Host Discovery table. You must assign roles and validate hardware before proceeding.

1. Verify all nodes appear in the **Host Discovery table**
2. Assign roles to each host: `control-plane` (master) or `worker`
3. Confirm hardware details — CPU, RAM, and disk must meet minimum requirements
4. Check NIC connectivity — the installer validates each node's network interface
5. Resolve any `error` or `insufficient` state before proceeding

> ⚠️ **NIC Issue Encountered:** Some nodes showed connectivity errors.
> **Fix:** Expand the host → deselect the inactive NIC → select the working NIC → re-validate.

---

## 7. Pre-Installation Validation

The Assisted Installer runs automatic checks before allowing installation to start. **All checks must be green.**

| Check | What to Look For |
|-------|-----------------|
| Host Status | All hosts show **"Ready"** state |
| API VIP | Reachable from all nodes |
| Ingress VIP | Reachable from all nodes |
| DNS Resolution | All required hostnames resolve correctly |
| NTP / Time Sync | Nodes can reach NTP servers |
| Disk Space | Min 200 GB free on masters, 120 GB on workers |
| IP Conflicts | No duplicate IPs in the network |
| Network Connectivity | All nodes can reach each other and the internet |

---

## 8. Starting the Installation

Once all validations pass, click **"Install Cluster"**. The cluster installs in phases automatically.

1. Click **"Install Cluster"** in the Assisted Installer UI
2. The installer writes RHCOS to each node's disk
3. Nodes reboot automatically into the installed OS
4. The bootstrap process initializes the API and control plane
5. Workers join the cluster once the control plane is ready
6. Operators initialize and the cluster becomes fully operational

### Installation Phases

| Phase | What Happens | Status to Watch |
|-------|-------------|----------------|
| Bootstrapping | Temporary bootstrap node starts the API server | API becomes reachable |
| Control Plane | Masters install RHCOS, etcd quorum forms | All 3 masters show Ready |
| Workers | Worker nodes install and join the cluster | Workers show Ready |
| Initialization | All operators initialize and become available | Progress bar reaches 100% |

---

## 9. Monitoring Progress

### From the UI
Watch the progress bar and the three status indicators — **Control Planes**, **Workers**, and **Initialization**.

### From the Bastion (CLI)

```bash
# Export the kubeconfig (download from UI or find in install dir)
export KUBECONFIG=/path/to/auth/kubeconfig

# Watch all nodes and cluster operators every 2 seconds ← most useful command
watch -n 2 'oc get no,co'

# Check overall cluster version and rollout progress
oc get clusterversion

# Check for non-running pods (useful when operators are stuck)
oc get pods -A | grep -v Running | grep -v Completed
```

### What to Expect

- **Initially:** most cluster operators show `AVAILABLE=False`, `PROGRESSING=True`
- Masters become `Ready` first, then workers join one by one
- Operators turn `AVAILABLE=True` progressively as the cluster stabilizes
- `authentication`, `console`, and `ingress` are among the **last to stabilize**
- **Final healthy state:** `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False` for all operators

> 💡 Brief `NotReady` periods for nodes are **normal** — especially during machine-config rollouts when nodes reboot to apply new configurations.

---

## 10. Issues Encountered & Resolutions

### Issue 1 — NIC Connectivity Error on Host Discovery

| | |
|--|--|
| **Symptom** | Node shows "Insufficient" or error state after booting from ISO |
| **Root Cause** | Wrong NIC was selected — the NIC was inactive or misconfigured |
| **Resolution** | Expand the host in the UI → deselect the inactive NIC → select the working NIC → click **Re-validate** |

---

### Issue 2 — SSH Access Failed Post-Boot

| | |
|--|--|
| **Symptom** | Could not SSH into the node from bastion after it booted from the ISO |
| **Root Cause** | Proxy certificate validation issue on the bastion side |
| **Resolution** | Workaround: access the node via the **VM console** directly until the proxy issue is resolved |

---

### Issue 3 — API Connection Timeout (`oc` commands failing)

| | |
|--|--|
| **Symptom** | "Unable to connect to server" when running `oc` commands on bastion |
| **Root Cause** | The Bastion VM was powered off |
| **Resolution** | Power on the Bastion VM. Connectivity restored immediately |

---

### Issue 4 — 1 Worker Failed During Initial Install (`ocp-worker3`)

| | |
|--|--|
| **Symptom** | UI showed "1/3 workers failed" during the installation process |
| **Root Cause** | Worker node failed mid-installation due to a disk or network timing issue |
| **Resolution** | Let the main cluster complete. Then use the **"Add Hosts"** tab to re-boot the failed worker with the discovery ISO and re-install it. See [Section 12](#12-adding-a-failed-worker-node) for the full procedure. |

---

### Issue 5 — Control Plane Nodes Briefly NotReady

| | |
|--|--|
| **Symptom** | `ocp-master2` and `ocp-master3` showed `NotReady` for a short period |
| **Root Cause** | Machine-config operator triggered node reboots after workers joined the cluster |
| **Resolution** | ✅ Self-resolving. Nodes returned to `Ready` status automatically within a few minutes |

---

### Issue 6 — Monitoring Operator Stuck Progressing

| | |
|--|--|
| **Symptom** | `clusteroperator/monitoring` showed `PROGRESSING=True` for an extended period |
| **Root Cause** | The Prometheus admission webhook deployment required 2 worker replicas, but workers were not yet `Ready` |
| **Resolution** | ✅ Self-resolving. The monitoring operator stabilized once both workers reached `Ready` state |

---

## 11. Post-Installation Verification

### 11.1 Final Node Status

```bash
[root@bastion-shivang ~]# oc get no
NAME                                        STATUS   ROLES                  AGE    VERSION
ocp-master1.shivang.lab.kuberox.net        Ready    control-plane,master   2d     v1.31.14
ocp-master2.shivang.lab.kuberox.net        Ready    control-plane,master   2d1h   v1.31.14
ocp-master3.shivang.lab.kuberox.net        Ready    control-plane,master   2d1h   v1.31.14
ocp-worker1.shivang.lab.kuberox.net        Ready    worker                 47h    v1.31.14
ocp-worker2.shivang.lab.kuberox.net        Ready    worker                 21m    v1.31.14
ocp-worker3.shivang.lab.kuberox.net        Ready    worker                 66m    v1.31.14
```

✅ All 6 nodes (3 masters + 3 workers) are in `Ready` state running Kubernetes `v1.31.14`.

### 11.2 Verify Cluster Operators

```bash
# All operators should be: AVAILABLE=True, PROGRESSING=False, DEGRADED=False
oc get co
```

### 11.3 Check Cluster Version

```bash
oc get clusterversion
```

### 11.4 Access the Web Console

- **URL:** `https://console-openshift-console.apps.shivang.lab.kuberox.net`
- **Login:** `kubeadmin` / password from the `kubeadmin-password` file
- **Download kubeconfig:** Use the **"Download kubeconfig"** button in the Assisted Installer UI

### 11.5 Approve Pending CSRs (if workers were added manually)

```bash
# Check for pending CSRs
oc get csr

# Approve all pending CSRs at once
oc get csr -o name | xargs oc adm certificate approve
```

---

## 12. Adding a Failed Worker Node Post-Installation

If a worker node failed during the initial installation, use this procedure to re-add it. The main cluster will already be running — this just adds the missing node.

> **Note:** The cluster installation shows "completed successfully" even if a worker failed. The failed worker is listed separately under *"Could not install 1 worker hosts"*.

1. Go to `console.redhat.com` → OpenShift → your cluster
2. Click the **"Add Hosts"** tab at the top of the cluster page
3. Click **"Add hosts"** button — this generates a new discovery ISO
4. Download the ISO and mount it on the failed worker VM via virtual media
5. Boot the worker node from the ISO
6. Wait for the node to appear in the **Host Discovery table** with `Ready` status
7. Click **"Install ready hosts"** — installation begins
8. Monitor progress: status changes from `Installing` → `Installed`
9. Verify the node joined the cluster:
   ```bash
   oc get nodes
   ```
10. Approve any pending CSRs if the node does not show `Ready`:
    ```bash
    oc get csr
    oc get csr -o name | xargs oc adm certificate approve
    ```

---

## 13. Deployment Timeline

| Time | Event |
|------|-------|
| 15:16 | ▶️ Installation started via Assisted Installer UI |
| 17:39 | ⚠️ Progress at 62% — 1 worker failed, 1 pending user action |
| 18:10 | Workers joining — `ocp-worker1` Ready, `ocp-worker2` NotReady |
| 18:26 | DNS degraded reported — `ocp-worker2` still NotReady |
| 18:29 | `ocp-master2`/`master3` briefly NotReady (machine-config triggered reboot) |
| 18:38 | Heavy operator activity — monitoring, kube-apiserver, OLM progressing |
| 19:03 | 5 nodes Ready (3 masters + 2 workers). All cluster operators Available |
| 19:05 | ✅ **Installation completed successfully** (confirmed in UI) |
| 20:27 | `ocp-worker3` added via Add Hosts tab — Installing 4/5 |
| 20:50 | ✅ `ocp-worker3` installed and joined cluster |
| ~2d later | ✅ All 6 nodes confirmed Ready (`oc get no` verified) |

---

## 14. Quick Reference Commands

| Command | What it Does |
|---------|-------------|
| `oc get nodes` | List all nodes and their Ready/NotReady status |
| `oc get co` | List all cluster operators with AVAILABLE/PROGRESSING/DEGRADED |
| `oc get clusterversion` | Show current cluster version and update status |
| `watch -n 2 'oc get no,co'` | **Live-refresh nodes + operators every 2 seconds** |
| `oc get csr` | Show pending certificate signing requests |
| `oc get csr -o name \| xargs oc adm certificate approve` | Approve all pending CSRs at once |
| `oc get pods -A \| grep -v Running \| grep -v Completed` | Find pods that are not running |
| `oc describe co <operator>` | Get detailed status and events for a specific operator |
| `oc logs -n <ns> <pod>` | View logs for a specific pod |
| `oc get events -A --sort-by=.lastTimestamp` | View all cluster events sorted by time |
| `oc debug node/<node>` | Open a debug shell on a specific node |

---

## 15. Key Learnings & Tips

### ✅ Always verify DNS first
Most installation failures trace back to DNS. Verify all records resolve correctly **before** clicking Install.

### ✅ Keep the Bastion running
The Bastion must stay **powered on throughout installation** for `oc` commands to work.

### ✅ Don't abort if a worker fails
A single worker failure does **not** stop the cluster. Let it complete and add the worker afterward using the Add Hosts flow.

### ✅ Brief NotReady periods are normal
Nodes reboot during machine-config rollouts. `NotReady` for 2–3 minutes is **expected and self-resolving**.

### ✅ Monitoring operator is last to stabilize
It needs both workers `Ready` before the Prometheus admission webhook can deploy.

### ✅ Download kubeconfig immediately
Save the `kubeconfig` and `kubeadmin-password` to a safe location right after installation completes.

### ✅ Use `watch -n 2 'oc get no,co'`
This single command gives you a complete real-time view of installation progress.

### ✅ Prepare IPs and MACs in advance
Document all static IPs and MAC addresses in a spreadsheet before starting — you'll need them during cluster config.

---

<div align="center">

**OpenShift 4.18 Assisted Installer Deployment Guide**

Shivang Bhardwaj ·

</div>
