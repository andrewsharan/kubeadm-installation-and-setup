# Kubeadm Cluster Setup

This guide provides an end-to-end blueprint for bootstrapping a multi-node Kubernetes cluster using **kubeadm on AWS EC2 instances** running **Ubuntu 24.04 LTS.**

**Bootstrapping** involves preparing the EC2 instances, installing the required Kubernetes components, initializing the control plane, joining the worker nodes to the cluster, configuring Calico networking, and bringing the cluster to a fully functional state.

## Table of Contents

1. [Architecture & Infrastructure Blueprint](#1-architecture--infrastructure-blueprint)
2. [AWS Prerequisites Configuration](#2-aws-prerequisites-configuration)
3. [Phase 1: Host OS Configuration (Run on All Nodes)](#3-phase-1-host-os-configuration-run-on-all-nodes)
4. [Phase 2: Container Runtime Installation (Run on All Nodes)](#4-phase-2-container-runtime-installation-run-on-all-nodes)
5. [Phase 3: Install Kubernetes Binaries (Run on All Nodes)](#5-phase-3-install-kubernetes-binaries-run-on-all-nodes)
6. [Phase 4: Control-Plane Bootstrap (Run Only on control-plane)](#6-phase-4-control-plane-bootstrap-run-only-on-control-plane)
7. [Phase 5: Install Calico CNI (Run Only on control-plane)](#7-phase-5-install-calico-cni-run-only-on-control-plane)
8. [Phase 6: Join Worker Nodes (Run on Worker Instances)](#8-phase-6-join-worker-nodes-run-on-worker-instances)
9. [Phase 7: Cluster Verification & End-to-End Testing](#9-phase-7-cluster-verification--end-to-end-testing)
10. [Troubleshooting Reference](#10-troubleshooting-reference)

---

## 1. Architecture & Infrastructure Blueprint

![AWS VPC architecture diagram showing the control-plane and two worker nodes secured by a self-referencing security group and connected over a Calico overlay network](kubeadm.png)

### Compute Sizing Requirements

| Node Role | Instance Type | vCPU | RAM | Root Volume (gp3) | Justification |
|---|---|---|---|---|---|
| **Control-Plane** | `t3.medium` | 2 | 4 GiB | 20 GiB | kubeadm preflight checks strictly enforce a minimum of 2 vCPUs and 1.7 GiB of RAM. Running etcd and the API server requires baseline compute to avoid crash-looping. |
| **Worker Node 1** | `t3.small` | 2 | 2 GiB | 20 GiB | Standard worker for application workloads. |
| **Worker Node 2** | `t3.small` | 2 | 2 GiB | 20 GiB | Standard worker for high-availability scheduling across zones. |

---

## 2. AWS Prerequisites Configuration

### IAM Instance Profile (SSM Access)

> **Security best practice:** Do not open inbound SSH (port 22) to the internet. Use AWS Systems Manager instead.

1. Create an AWS IAM role with the **AmazonSSMManagedInstanceCore** policy.
2. Attach this IAM role as an **instance profile** to all three EC2 instances.
3. Access nodes securely via **AWS Systems Manager (Session Manager)** from the AWS Console or the AWS CLI (`aws ssm start-session`).

### Security Group Inbound Rules

Attach a single common security group to all three nodes with the following inbound rules:

| Type | Protocol | Port Range | Source | Justification |
|---|---|---|---|---|
| All traffic | All | All | `sg-0817441aefccd013c` (self-referencing SG ID) | **Crucial:** Allows unrestricted pod-to-pod overlay network traffic (VXLAN 4789 / IP-in-IP protocol 4), CoreDNS resolution (port 53), etcd peer sync (2379–2380), and kubelet execution tunneling (10250) across the cluster nodes. |
| Custom TCP | TCP | 30000–32767 | `<Your-Public-IP>/32` | Allows access to exposed Kubernetes NodePort application services from your local browser without opening them to the entire internet. |
| All ICMP - IPv4 | ICMP | All | `172.31.0.0/16` | Enables internal connectivity testing (ping) within the VPC. |

**Outbound rules:** Keep the default **All Traffic (`0.0.0.0/0`)** rule to permit image pulls and repository package downloads.

**Note:** Since this is a lab setup, I used an **“All traffic”** rule with a self-referencing security group to simplify node-to-node communication and ensure Kubernetes/Calico networking works correctly. For a production environment, **separate Security Groups should be created for the Kubernetes control plane and worker nodes.** The inbound and outbound rules should then be restricted according to the principle of least privilege, allowing only the required protocols and port numbers between the appropriate cluster components. This provides better network isolation and reduces the potential attack surface.


## 3. Phase 1: Host OS Configuration

Perform these steps on `control-plane`, `worker-node1`, and `worker-node2`.

### Step 1: Set Hostnames and Update /etc/hosts

Kubernetes requires resolvable, predictable hostnames for certificate Subject Alternative Names (SANs) and node registration.

**On control-plane:**
```bash
sudo hostnamectl set-hostname control-plane
echo "$(hostname -I | awk '{print $1}') control-plane" | sudo tee -a /etc/hosts
```

**On worker-node1:**
```bash
sudo hostnamectl set-hostname worker-node1
echo "$(hostname -I | awk '{print $1}') worker-node1" | sudo tee -a /etc/hosts
```

**On worker-node2:**
```bash
sudo hostnamectl set-hostname worker-node2
echo "$(hostname -I | awk '{print $1}') worker-node2" | sudo tee -a /etc/hosts
```

**What this does:**
- `hostnamectl set-hostname` — Sets the host system identity.
- `tee -a /etc/hosts` — Maps the private IP directly to the hostname locally, bypassing potential external DNS resolution failures with the local resolver (`127.0.0.53`).
- **Verification:** — Run `cat /etc/hosts ` or `getent hosts control-plane`.  The output should display the IP address along with the hostname.

### Step 2: Disable Linux Swap Memory

The Kubernetes kubelet does not support swap allocation, because swap breaks container memory isolation and QoS classifications.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

**What this does:**
- `swapoff -a` — Disables swap in active kernel memory immediately.
- `sed -i ... /etc/fstab` — Permanently comments out any swap partition or file so swap remains disabled after reboots.
- **Verification:** Run `free -m`. The `Swap:` line must read `0` across all columns.

### Step 3: Load Linux Networking Kernel Modules

Containers communicate across virtual interfaces. The Linux kernel requires specific modules to handle network overlays and packet filtering.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

**What this does:**
- `overlay` — Enables the storage driver to merge container layers using OverlayFS.
- `br_netfilter` — Forces bridged network packets across virtual interfaces to be processed by host iptables rules.
- **Verification:** Run `lsmod | grep -E 'overlay|br_netfilter'`. Both modules must appear in the output.

### Step 4: Configure Kernel Sysctl Parameters

Configure the host to route internal packets and pass them to netfilter chains.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**What this does:**
- `net.ipv4.ip_forward = 1` — Converts the host into a packet router. Without this, the Linux kernel drops packets destined for container IP ranges outside its own interface.
- `net.bridge.bridge-nf-call-iptables = 1` — Ensures bridged IPv4 packets pass through host netfilter/iptables rules.
- **Verification:** Run `sysctl net.ipv4.ip_forward net.bridge.bridge-nf-call-iptables`. Both values must equal `1`.

### Step 5: Install Low-Level Network Dependencies

Kubernetes node agents require tools to manage connection tracking and dynamic virtual routing.

```bash
sudo apt-get update
sudo apt-get install -y conntrack socat ipset
```

**What this does:**
- `conntrack` — Manages netfilter stateful connection tracking. Required by kube-proxy to clear stale NAT sessions when pods terminate.
- `socat` — A bidirectional relay utility used by kubelet to support commands like `kubectl port-forward`.
- `ipset` — Allows kube-proxy and Calico to store firewall rules in O(1) lookup hash tables instead of linear O(N) iptables chains.
- **Verification:** Run `which conntrack socat ipset`. All three paths should be displayed.

---

## 4. Phase 2: Container Runtime Installation

Perform these steps on `control-plane`, `worker-node1`, and `worker-node2`.

Kubernetes requires an OCI-compliant Container Runtime Interface (CRI). This guide uses standard `containerd`.

### Step 1: Install containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd
```

### Step 2: Configure the systemd cgroup Driver

By default, containerd initializes with the legacy `cgroupfs` driver. Ubuntu uses `systemd` to manage cgroups — running mismatched drivers causes kernel panics under resource contention.

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**What this does:**
- `containerd config default` — Outputs the fully populated default configuration.
- `sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g'` — Instructs containerd to use `systemd` as the control group driver for container resource slicing.
- **Verification:** Run `systemctl status containerd`. The service must be `active (running)`.

---

## 5. Phase 3: Install Kubernetes Binaries

Perform these steps on `control-plane`, `worker-node1`, and `worker-node2`.

Install the target version of Kubernetes (v1.31) from the official package repository ([pkgs.k8s.io](http://pkgs.k8s.io)).

### Step 1: Add the Official Kubernetes APT Repository

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Step 2: Install and Hold the Packages

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**What this does:**
- `apt-mark hold` — Locks package versions, preventing accidental automatic updates during operating system patches that could break cluster version compatibility.
- **Verification:** Run `kubeadm version && kubelet --version`. Both should return `v1.31.x`.

---

## 6. Phase 4: Control-Plane Bootstrap 

Perform these steps on `control-plane`.

### Step 1: Initialize the Cluster with kubeadm

Execute the cluster initialization on the control-plane instance:

```bash
IP_ADDR=$(hostname -I | awk '{print $1}')

sudo kubeadm init \
  --apiserver-advertise-address=$IP_ADDR \
  --pod-network-cidr=192.168.0.0/16 \
  --node-name control-plane
```

**What this does:**
- `--apiserver-advertise-address` — The private IP of your control-plane node (e.g. `172.31.91.187`). Worker nodes reach the API server on this address.
- `--pod-network-cidr=192.168.0.0/16` — Reserves an IP pool specifically for pods across all nodes (matches the Calico defaults used below).
- `--node-name control-plane` — Matches the internal Kubernetes node object name directly to the OS hostname.

**What kubeadm executes during this step:**
1. Generates the root CA and TLS certificates in `/etc/kubernetes/pki/`.
2. Generates administrative and component kubeconfig files in `/etc/kubernetes/`.
3. Writes static pod manifests for `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, and `etcd` into `/etc/kubernetes/manifests/`.
4. The local kubelet detects the manifests and starts the control-plane containers.
5. Deploys cluster add-ons: CoreDNS and kube-proxy.

### Step 2: Configure kubectl Access

Provide your active user account with the credentials needed to interact with the API server:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Verification:** Run `kubectl get nodes`. You should see:

```
NAME            STATUS     ROLES           AGE   VERSION
control-plane   NotReady   control-plane   1m    v1.31.14
```

> **Note:** The status remains `NotReady` until the CNI plugin is installed in Phase 5.

### Step 3: Copy the Join Command

The output of `kubeadm init` concludes with a join token string. Save this line for the worker nodes:

```bash
kubeadm join 172.31.91.187:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 7. Phase 5: Install Calico CNI 

Perform these steps on `control-plane`.

Vanilla Kubernetes has no built-in pod network driver; it requires a CNI to allocate IPs and route packets between nodes.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

**What this does:**
- `tigera-operator.yaml` — Deploys the controller that manages the Calico lifecycle.
- `custom-resources.yaml` — Provisions the IP pool matching your `--pod-network-cidr=192.168.0.0/16` specification.

**Verification:** Run `watch kubectl get nodes`. Within 30–60 seconds, the control-plane status flips from `NotReady` to **`Ready`**. Press `Ctrl+C` to exit.

---

## 8. Phase 6: Join Worker Nodes

Perform these steps on `worker-node1` and `worker-node2`.

Log into each worker node via Session Manager and run the join command generated during Phase 4 with `sudo`, appending the explicit `--node-name` parameter:

**On worker-node1:**
```bash
sudo kubeadm join 172.31.91.187:6443 --token <your-token> \
    --discovery-token-ca-cert-hash sha256:<your-hash> \
    --node-name worker-node1
```

**On worker-node2:**
```bash
sudo kubeadm join 172.31.91.187:6443 --token <your-token> \
    --discovery-token-ca-cert-hash sha256:<your-hash> \
    --node-name worker-node2
```

> **Note:** If your token expires or is lost, regenerate a new join command on the control-plane with `kubeadm token create --print-join-command`.

---

## 9. Phase 7: Cluster Verification & End-to-End Testing

Return to the **control-plane** terminal.

### 1. Verify Node Registration

```bash
kubectl get nodes -o wide
```

**Expected output:**
```
NAME            STATUS   ROLES           AGE   VERSION    INTERNAL-IP     OS-IMAGE             CONTAINER-RUNTIME
control-plane   Ready    control-plane   10m   v1.31.14   172.31.91.187   Ubuntu 24.04.4 LTS   containerd://2.2.1
worker-node1    Ready    <none>          3m    v1.31.14   172.31.67.162   Ubuntu 24.04.4 LTS   containerd://2.2.1
worker-node2    Ready    <none>          2m    v1.31.14   172.31.69.237   Ubuntu 24.04.4 LTS   containerd://2.2.1
```

### 2. Verify System Pods

```bash
kubectl get pods -A
```

Ensure all pods in `kube-system`, `calico-system`, and `tigera-operator` report a `Running` status with zero restarts.

### 3. Test Cross-Node Network Connectivity

Deploy a 2-replica test workload across the worker nodes:

```bash
kubectl create deployment net-test --image=busybox --replicas=2 -- sleep 3600
kubectl get pods -o wide
```

Identify the pod name on `worker-node1` and the IP address of the pod scheduled on `worker-node2`:

```bash
POD_1=$(kubectl get pods -l app=net-test -o jsonpath='{.items[0].metadata.name}')
POD_2_IP=$(kubectl get pods -l app=net-test -o jsonpath='{.items[1].status.podIP}')

# Ping across the overlay network
kubectl exec -it $POD_1 -- ping -c 3 $POD_2_IP
```

**Verification:** Must show **0% packet loss**.

### 4. Test In-Cluster DNS Resolution

```bash
kubectl expose deployment net-test --port=80 --target-port=8080
kubectl exec -it $POD_1 -- nslookup net-test
```

**Verification:** CoreDNS returns the virtual `ClusterIP` of the `net-test` service.

Clean up the test resources:

```bash
kubectl delete service net-test
kubectl delete deployment net-test
```

---

## 10. Troubleshooting Reference

| Symptom | Cause | Solution |
|---|---|---|
| `[ERROR FileExisting-conntrack]: conntrack not found in system path` | The `conntrack` utility was omitted during package setup. | Run `sudo apt-get install -y conntrack socat ipset` on the affected node. |
| `[WARNING Hostname]: hostname "..." could not be reached` | Hostname is not resolvable by the local stub resolver (`127.0.0.53`). | Add `<Private-IP> <hostname>` directly to `/etc/hosts`. |
| Node remains in `NotReady` state | The CNI plugin is not installed, or the pod CIDR does not match the CNI configuration. | Ensure the Calico operator and custom resources are applied, and verify `--pod-network-cidr` matches `custom-resources.yaml`. |
| Pod ping across nodes fails (100% loss) | The AWS security group is dropping encapsulated overlay packets. | Add an **All Traffic** rule pointing to the security group ID itself (`sg-...`) to permit east-west traffic. |
| `The connection to the server localhost:8080 was refused` | `kubectl` cannot locate `~/.kube/config`. | Run `export KUBECONFIG=/etc/kubernetes/admin.conf`, or ensure `$HOME/.kube/config` exists and is readable. |
