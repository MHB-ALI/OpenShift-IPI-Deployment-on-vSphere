# Deployment Steps — OpenShift IPI Installation on vSphere

This guide walks through deploying **Red Hat OpenShift Container Platform (OCP) 4.18** on VMware vSphere using the **Installer-Provisioned Infrastructure (IPI)** method. IPI is the "hands-off" installation path — once the prerequisites below are met and `install-config.yaml` is generated, the installer provisions the VMs, networking, and control plane itself.

> Make sure you've completed the [Prerequisites](./01-prerequisites.md) before starting — this deployment assumes vCenter, DNS, DHCP/IP addressing, and network ports are already in place.

---

## Step 1 — Deploy vCenter Server

Deploy (or have access to) a VMware **vCenter Server** instance that meets the supported version requirements.

**Why vCenter is required:** IPI installation isn't just deploying OpenShift *onto* existing VMs — the installer talks directly to the vCenter API to create the folder, tags, templates, and VMs that make up the cluster, and continues to use that API afterward for dynamic storage provisioning (via the CSI driver) and node lifecycle management. Without vCenter, the installer has no infrastructure to provision against.

---

## Step 2 — Prepare a Red Hat Helper Machine

Stand up a RHEL (or RHEL-compatible) VM to act as your **installation helper/bastion host** — referred to as `oc-helper` throughout this guide.

**Why a helper machine is needed:** The OpenShift installer binary, `oc` CLI, SSH keys, and vCenter trust certificates all need to live somewhere with network access to both vCenter and the cluster's API/Ingress VIPs. Rather than running the installer from your laptop, a dedicated Linux helper machine keeps credentials and installation artifacts (like `install-config.yaml`, which contains your vCenter credentials and pull secret) in a controlled, reusable location — especially useful if you need to re-run or destroy/recreate the cluster later.

---

## Step 3 — Create DNS Records

Create the two required DNS records identified during prerequisites:

```
api.<cluster_name>.<base_domain>       → api.ocp418.testlab.local
*.apps.<cluster_name>.<base_domain>    → *.apps.ocp418.testlab.local
```

**What these records are for:**
- `api.<cluster_name>.<base_domain>` — resolves to the **API VIP**, used by `oc`, the installer, and any external tooling to reach the Kubernetes API server.
- `*.apps.<cluster_name>.<base_domain>` — a wildcard resolving to the **Ingress VIP**, used to route traffic to every route/application exposed through the OpenShift router (e.g., the web console, and any app you deploy).

Both records must resolve correctly from **outside** the cluster (your workstation/network) and from **inside** the cluster (the nodes themselves), or installation will hang or fail at bootstrap.

---

## Step 4 — Configure DHCP (or Static IPs)

Configure a DHCP server (Windows DHCP role, `dnsmasq`, `isc-dhcp-server`, etc.) — or plan static IP assignments if you prefer not to use DHCP.

**Minimum IP requirement:** You need **at least 6 DHCP-assigned IP addresses** for a standard cluster:

| Node | Count |
|---|---|
| Bootstrap (temporary) | 1 |
| Control plane | 3 |
| Compute/worker | 2 (minimum) |

*(Plus the 2 static VIPs — API and Ingress — covered in prerequisites, which are separate from the DHCP pool.)*

---

## Step 5 — Download the OpenShift Installer and CLI

1. Log in to (or create) a **Red Hat account**.
2. Access the **Red Hat Hybrid Cloud Console**.
3. Navigate to: **OpenShift → Cluster List → Cluster Type: VMware vSphere → Installer-provisioned infrastructure**.
4. Under **What you need to get started**:
   - Select **Linux** and click **Download installer** (the OpenShift installer binary — `openshift-install`).
   - Select **Linux – RHEL 9** and click **Download command-line tools** to get the `oc` client.
   - Note the **Pull secret** section here too — you'll need it in Step 9.

![Red Hat Hybrid Cloud Console – download installer and CLI](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a309d86a8e02be72c2145f39ac1c607f5a40506f/Images/Step%205%20-%20Openshift%20Hybrid%20Cloud%20Console%20-%20Openshift%20Installer%20Download.jpg)

---

## Step 6 — Generate SSH Keys and Trust the vCenter CA

### 6.1 Download the vCenter trusted root CA

Open your vCenter home page, right-click **Download trusted root CA certificates**, and choose **Save Link As**. This downloads a `download.zip` file.

![Downloading the vCenter trusted root CA certificate](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/dba8e40d54bfef56aa6a0bdb2be1247cf30b8450/Images/Step%206%20-%20vCenter%20Trusted%20Root%20CA%20Download.jpg)

Upload `download.zip` to your `oc-helper` Linux machine, then unzip it:

```bash
unzip download.zip
```

This produces a `certs/` directory with subfolders for each OS (`lin`, `mac`, `win`):

![Unzipped CA certificate directory structure](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/81ea0f19f5f68a42e3a4e293a7941afd7410f92a/Images/Step%206%20-%20vCenter%20Cert%20Folder.png)

> **Why this matters:** Because the installation program communicates with your vCenter's API over HTTPS, you must add vCenter's trusted root CA certificates to your Linux system's certificate trust store *before* installing OpenShift — otherwise the installer will fail to validate the connection to vCenter.

### 6.2 Generate an SSH key pair

This key pair lets you authenticate to the OpenShift cluster's nodes after deployment (useful for troubleshooting).

```bash
ssh-keygen
```

![Generating the SSH key pair](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/81ea0f19f5f68a42e3a4e293a7941afd7410f92a/Images/Step%206%20-%20Generate%20Local%20Certificate.jpg)

Accept the defaults (saves to `~/.ssh/id_rsa` / `~/.ssh/id_rsa.pub`) unless you have a reason to use a custom path.

### 6.3 Add the vCenter CA certificates to the system trust store

Copy the Linux CA certs into the trust anchors directory:

```bash
cp certs/lin/* /etc/pki/ca-trust/source/anchors/
```

![Copying CA certificates to the trust anchors directory](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/81ea0f19f5f68a42e3a4e293a7941afd7410f92a/Images/Step%206%20-%20Copy%20Linux%20Certificate.png)

Then update the system trust store:

```bash
update-ca-trust extract
```

![Updating the system CA trust store](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/81ea0f19f5f68a42e3a4e293a7941afd7410f92a/Images/Step%206%20-%20Update%20CA%20Trust.png)

Running this command establishes trust between your `oc-helper` machine and vCenter, which the installer relies on for all subsequent API calls.

---

## Step 7 — Create a Working Directory

Create a dedicated directory to hold the installer, CLI, and generated cluster configuration/state files:

```bash
mkdir ocp418
```

Keeping everything in one directory matters — the installer writes cluster state (including credentials to tear the cluster down later) into this directory, so it should not be deleted after installation.

---

## Step 8 — Install the OpenShift CLI and Installer Binaries

Upload the downloaded installer and client archives into the `ocp418` directory:

![Installer and client archives uploaded to the working directory](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a92914de3ddccb819088a522ce4321cafa3df31f/Images/Step%208%20-%20Upload%20to%20directory.jpg)

### 8.1 Extract and install the `oc` client

```bash
tar xvf openshift-client-linux-amd64-rhel9.tar.gz
```

![Extracting the OpenShift client](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a92914de3ddccb819088a522ce4321cafa3df31f/Images/Step%208%20-%20Extract%20Openshift%20Client.jpg)

Move `oc` and `kubectl` to a directory on your `PATH` (per the bundled `README.md`):

```bash
mv oc kubectl /usr/local/bin
```

### 8.2 Extract and install the OpenShift installer

```bash
tar xvf openshift-install-linux.tar.gz
```

Move the `openshift-install` binary to the same location:

```bash
cp openshift-install /usr/local/bin
```

![Extracting and installing the openshift-install binary](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a92914de3ddccb819088a522ce4321cafa3df31f/Images/Step%208%20-%20Copy%20Openshift%20Installer.jpg)

---

## Step 9 — Create the Install Config and Deploy the Cluster

### 9.1 Generate `install-config.yaml`

```bash
openshift-install create install-config --dir=ocp418 --log-level=info
```

The installer runs an interactive wizard and prompts for:

| Prompt | Example value |
|---|---|
| SSH Public Key | `/root/.ssh/id_rsa.pub` |
| Platform | `vsphere` |
| vCenter | `vcsa.testlab.local` |
| Username | `administrator@vsphere.local` |
| Password | *(hidden input)* |
| Datacenter | `TESTLAB-DC` *(auto-detected if only one exists)* |
| Cluster (compute cluster) | `/TESTLAB-DC/host/esxi01-testlab` *(auto-detected if only one exists)* |
| Default Datastore | `/TESTLAB-DC/datastore/Unity-DS1` |
| Network | `VM Network` |
| Virtual IP Address for API | `192.x.x.x` |
| Virtual IP Address for Ingress | `192.x.x.x` |
| Base Domain | `testlab.local` |
| Cluster Name | `ocp418` |
| Pull Secret | *(pasted — see below)* |

![install-config.yaml wizard – vCenter and network prompts](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/524b0e2f207ed5b2c6d03dd02379c7dbfd345f2f/Images/Step%209%20-%20Create%20Install-Config%20YAML.png)

For the **Pull Secret** prompt, go back to the Hybrid Cloud Console page from Step 5 and click **Copy pull secret**:

![Copying the pull secret from the Hybrid Cloud Console](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a92914de3ddccb819088a522ce4321cafa3df31f/Images/Step%209%20-%20Copy%20Secret.jpg)

Paste it in when prompted:

![Pasting the pull secret to complete the install-config wizard](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a92914de3ddccb819088a522ce4321cafa3df31f/Images/Step%209%20-%20Paste%20Secret.jpg)

Once complete, the installer writes `install-config.yaml` into the `ocp418` directory:

![install-config.yaml generated in the working directory](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a92914de3ddccb819088a522ce4321cafa3df31f/Images/Step%209%20-%20Config%20YAML%20Created.jpg)

You can review the generated file at any time with:

```bash
cat install-config.yaml
```

> **Tip:** `install-config.yaml` is consumed (and deleted) on the next step, so keep a backup copy if you want to reuse these settings for a future cluster.

### 9.2 Create the cluster

From the working directory:

```bash
openshift-install create cluster --dir=ocp418 --log-level=info
```

Or, if you're already inside the `ocp418` directory:

```bash
openshift-install create cluster .
```

The installer will now provision the bootstrap node, control plane, and compute nodes in vCenter, and bring up the cluster. This typically takes **around 45 minutes**.

![openshift-install create cluster in progress](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/a92914de3ddccb819088a522ce4321cafa3df31f/Images/Step%209%20-%20Deploy%20Openshift%20Cluster.jpg)

On success, the installer prints the cluster's console URL and `kubeadmin` credentials, and writes a `kubeconfig` file you can use with `oc` to access the cluster.

---

## Cleaning Up a Failed Deployment

If a deployment fails partway through, don't just re-run `create cluster` — leftover vCenter resources (VMs, folders, tags) from the failed attempt can conflict with a fresh install. Destroy the cluster first:

```bash
openshift-install destroy cluster --dir=/root/oc-helper/ocp418
```

This removes all vCenter resources the installer created, after which you can correct your `install-config.yaml` and retry `create cluster`.

---

## Summary

| Step | Action |
|---|---|
| 1 | Deploy vCenter Server |
| 2 | Prepare a Red Hat helper (bastion) machine |
| 3 | Create API and Ingress DNS records |
| 4 | Configure DHCP (min. 6 IPs) or static IP plan |
| 5 | Download installer, `oc` CLI, and pull secret |
| 6 | Generate SSH keys and trust the vCenter CA |
| 7 | Create a working directory |
| 8 | Extract and install the CLI/installer binaries |
| 9 | Generate `install-config.yaml` and run `create cluster` |

*Previous: [Prerequisites](https://github.com/MHB-ALI/OpenShift-IPI-Deployment-on-vSphere/blob/179a90a1811ee46da78c7b4411ec309f27c08e5d/Prerequisites/prerequisites.md)*
