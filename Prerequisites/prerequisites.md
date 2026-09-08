# Red Hat OpenShift Container Platform — IPI Installation on VMware vSphere

## Prerequisites

This guide covers the prerequisites for deploying a Red Hat OpenShift Container Platform (OCP) cluster on VMware vSphere using the **Installer-Provisioned Infrastructure (IPI)** method. IPI automates the provisioning of the underlying vSphere infrastructure (VMs, networking, storage) as part of the OpenShift installation, so getting these prerequisites right up front is critical for a smooth deployment.

---

### 1. Supported vSphere Platform Version

Your VMware environment must be running one of the following supported versions:

| Platform | Supported Version |
|---|---|
| VMware vSphere | 8.0 Update 1 or later |
| VMware Cloud Foundation | 5.0 or later |
| VMware vSphere Foundation | 9 or later |
| VMware Cloud Foundation | 9 or later |

> **Note:** A vCenter Server instance is **mandatory** for OCP deployment. IPI installation is not supported against a standalone ESXi host — the installer communicates with vCenter to provision and manage cluster resources.

---

### 2. vCenter Account Permissions

The OpenShift installation program needs an account with sufficient privileges to create and manage resources in vCenter (folders, VMs, tags, networks, etc.).

You have two options:

1. **Global administrator account** (simplest) — grants all necessary permissions automatically and avoids the need to hand-build custom roles.
2. **Custom vSphere role(s)** — if a global admin account isn't available, you must create one or more custom roles with the specific privileges OpenShift requires, and assign them to the relevant vCenter objects (vCenter, datacenter, datastore, cluster/host, network, folder).

   - Most privileges are **always required**.
   - Some privileges are only required if you let the installer **auto-provision a folder** for the cluster (this is the default behavior).

📖 Full privilege list: [Required vCenter account privileges — Red Hat OCP 4.18 docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_on_vmware_vsphere/installer-provisioned-infrastructure#installation-vsphere-installer-infra-requirements-account_ipi-vsphere-installation-reqs)

---

### 3. Time Synchronization (NTP)

All ESXi hosts in the cluster must have synchronized clocks **before** installation begins.

- Configure an NTP server on each ESXi host.
- Clock drift between hosts can cause certificate validation errors and cluster instability during and after installation, so this step should not be skipped even in lab/test environments.

---

### 4. Required Network Ports

OpenShift cluster components communicate over a specific set of ports. Your network (firewalls, security groups, NSX rules, etc.) must allow this traffic between all cluster nodes.

#### Host-level and general cluster communication

| Protocol | Port(s) | Purpose |
|---|---|---|
| VRRP | N/A | Required for keepalived (API/Ingress VIP failover) |
| ICMP | N/A | Network reachability tests |
| TCP | 1936 | Metrics |
| TCP | 9000–9999 | Host-level services — includes node exporter (9100–9101) and Cluster Version Operator (9099) |
| TCP | 10250–10259 | Default ports reserved by Kubernetes |
| UDP | 6081 | Geneve (overlay networking) |
| UDP | 9000–9999 | Host-level services, including node exporter (9100–9101) |
| UDP | 500 | IPsec IKE packets |
| UDP | 4500 | IPsec NAT-T packets |
| TCP/UDP | 30000–32767 | Kubernetes NodePort services |
| ESP | N/A | Kubernetes node-to-node IPsec traffic |

#### All-machine to control plane communication

| Protocol | Port | Description |
|---|---|---|
| TCP | 6443 | Kubernetes API |

#### Control plane to control plane communication

| Protocol | Port | Description |
|---|---|---|
| TCP | 2379–2380 | etcd server and peer ports |

---

### 5. vSphere CSI Driver Requirements

The vSphere Container Storage Interface (CSI) Driver Operator enables OpenShift to dynamically provision persistent storage backed by vSphere. It has its own set of minimum requirements:

- VMware vSphere **8.0 Update 1** or later, **or** VMware vSphere Foundation (VVF) **9**, **or** VMware Cloud Foundation (VCF) **5** or later
- vCenter Server **8.0 Update 1** or later, **or** VVF 9, **or** VCF 5 or later
- Virtual machine **hardware version 15** or later
- **No third-party vSphere CSI driver** already installed in the cluster (conflicts with the built-in operator)

---

### 6. Cluster Resource Footprint

During installation, the IPI installer automatically creates the following objects in vCenter:

- **1** Folder
- **1** Tag category
- **1** Tag
- Virtual machines:
  - **1** template (used to clone the other VMs)
  - **1** temporary bootstrap node *(deleted automatically once the control plane is up)*
  - **3** control plane (master) nodes
  - **3** compute (worker) nodes

**Storage sizing:**

| Item | Storage |
|---|---|
| Total provisioned during install (incl. bootstrap) | ~856 GB |
| Minimum required for a standard cluster (post-bootstrap) | 800 GB |

> Adding more compute nodes beyond the default 3 will increase total storage consumption accordingly — plan datastore capacity ahead of time.

---

### 7. Networking and DHCP

- OpenShift on vSphere IPI supports **DHCP** for automatic IP assignment, or **static IP addresses** if you prefer not to use DHCP.
- If using DHCP, the DHCP server **must** be configured to hand out a default gateway in the lease.
- It's recommended (and best practice) that each node can reach an **NTP server discoverable via DHCP**. While the installation can technically succeed without one, unsynchronized node clocks can lead to subtle and hard-to-diagnose errors later.
- If deploying into a **restricted/disconnected network**, the VM(s) in that network still need network access to **vCenter** — the installer relies on this connectivity to provision and manage nodes, PVCs, and other resources throughout the cluster's lifecycle, not just during initial install.

---

### 8. Required IP Addresses

For a DHCP-based network, the installer still requires **two static/reserved IP addresses** to be provided at install time:

| Address | Purpose |
|---|---|
| **API VIP** | Used for accessing the cluster's Kubernetes API |
| **Ingress VIP** | Used for routing external traffic into the cluster (apps/routes) |

These are supplied as install-config parameters and are not assigned dynamically by DHCP.

---

### 9. Required DNS Records

Before installation, you must pre-create the following DNS records. Both must be resolvable **from outside the cluster** and **from within the cluster** (i.e., by the nodes themselves).

| Component | Record | Description |
|---|---|---|
| **API VIP** | `api.<cluster_name>.<base_domain>` | A/AAAA or CNAME record pointing to the load balancer in front of the control plane machines. |
| **Ingress VIP** | `*.apps.<cluster_name>.<base_domain>` | Wildcard A/AAAA or CNAME record pointing to the load balancer targeting the Ingress router pods (worker nodes, by default). |

> Replace `<cluster_name>` and `<base_domain>` with the values you plan to use in your `install-config.yaml`.

---

### ✅ Prerequisites Checklist

- [ ] vCenter running a supported vSphere/VCF/VVF version
- [ ] vCenter account with global admin **or** custom roles with required privileges
- [ ] NTP configured and synchronized across all ESXi hosts
- [ ] All required network ports open between nodes
- [ ] vSphere CSI driver requirements met (hardware version 15+, no conflicting CSI driver)
- [ ] Sufficient datastore capacity (≥800 GB, more if scaling compute nodes)
- [ ] DHCP configured with default gateway **or** static IP plan in place
- [ ] Two reserved IP addresses available (API VIP, Ingress VIP)
- [ ] DNS records created for API and Ingress (internally and externally resolvable)

---

*Next: [Deployment Steps](./02-deployment.md) — installing OpenShift Container Platform on vSphere using the IPI method.*
