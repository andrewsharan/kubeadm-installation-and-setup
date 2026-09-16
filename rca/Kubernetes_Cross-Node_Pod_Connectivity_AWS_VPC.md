# Root Cause Analysis: Kubernetes Cross-Node Pod Connectivity Failure on AWS VPC

## 1. Executive Summary

During operational testing of a newly provisioned Kubernetes cluster hosted on AWS EC2 instances running Ubuntu 24.04.4 LTS, cross-node Pod-to-Pod communication failed completely (100% packet loss).

| Category | Details |
|---|---|
| **Affected Workload** | Pod `net-test-79c969fccd-pkjl9` (IP: `192.168.180.195`), scheduled on `worker-node1`, was unable to communicate with Pod `net-test-79c969fccd-s6rsm` (IP: `192.168.203.131`), scheduled on `worker-node2`. |
| **Symptom** | ICMP echo requests issued from inside the Pod network namespace on `worker-node1` to the remote Pod IP resulted in a complete timeout (see command below). |
| **Cluster State** | Node-level communication, Kubernetes control plane status, container runtime health, and local container networking (intra-node communication) all functioned normally. |
| **Root Cause** | The cluster's CNI plugin (Project Calico) was configured with an IP Pool encapsulation profile of `vxlanMode: CrossSubnet`. Because both worker nodes resided in the same AWS VPC IPv4 CIDR subnet (`172.31.64.0/20`), Calico dynamically suppressed overlay encapsulation (VXLAN) in favor of native Linux L3 direct host routing to optimize throughput. As a result, the EC2 instances transmitted unencapsulated packets using private Pod IPs as the outer IP header's source and destination addresses. The AWS VPC fabric dropped these packets at the hypervisor layer because the Elastic Network Interfaces (ENIs) had the AWS default **Source/Destination Check** enabled (`SourceDestCheck: true`). |
| **Resolution** | Disabling `SourceDestCheck` on the underlying AWS EC2 Elastic Network Interfaces allowed the AWS VPC substrate to permit transit packets whose network-layer headers did not match the instances' assigned private IPs. Cross-node network throughput was restored instantly, with 0% packet loss and a round-trip latency of ~0.337 ms. |

**Symptom command:**

```bash
kubectl exec -it net-test-79c969fccd-pkjl9 -- ping -c 3 192.168.203.131
# Result: 3 packets transmitted, 0 packets received, 100% packet loss
```

---

## 2. Infrastructure & Architectural Topology

The cluster ran on a three-node vanilla topology orchestrated on top of Amazon Web Services (AWS) Elastic Compute Cloud (EC2) instances.

```text
+------------------------------------------------------------+
|                      AWS VPC (Region)                      |
|              Subnet: 172.31.64.0/20 (us-east-1a)            |
+------------------------------------------------------------+
                               |
        +--------------------------------+   +--------------------------------+
        |  worker-node1 (EC2 Instance)   |   |  worker-node2 (EC2 Instance)   |
        |  Instance ID:                  |   |  Instance ID:                  |
        |    i-0281a0d1ab953427e         |   |    i-0eb935deffa4b32fb         |
        |  Host/ENI IP: 172.31.67.162    |   |  Host/ENI IP: 172.31.69.237    |
        |  Pod CIDR: 192.168.180.192/26  |   |  Pod CIDR: 192.168.203.128/26  |
        +--------------------------------+   +--------------------------------+
        |  Pod: net-test-***-pkjl9       |   |  Pod: net-test-***-s6rsm       |
        |  IP: 192.168.180.195           |   |  IP: 192.168.203.131           |
        |  Veth: cali1a2b3c...           |   |  Veth: cali9z8y7x...           |
        +--------------------------------+   +--------------------------------+
```

### Component Breakdown

| Component | Value |
|---|---|
| Kubernetes Control Plane | `v1.31.14` |
| Operating System Base | Ubuntu 24.04.4 LTS (Noble Numbat) |
| Linux Kernel | `6.17.0-1017-aws` (x86_64) |
| Container Runtime Interface (CRI) | `containerd://2.2.1` |
| Container Network Interface (CNI) | Project Calico v3.28.x / Tigera Operator |
| Network Driver | Amazon ENA (Elastic Network Adapter), bound to interface `ens5` |

**AWS Addressing**

| Node | Role | Private IP | Instance ID |
|---|---|---|---|
| Control plane node | Control plane | `172.31.91.187` | `i-066195c81368451a7` |
| worker-node1 | Worker | `172.31.67.162` | `i-0281a0d1ab953427e` |
| worker-node2 | Worker | `172.31.69.237` | `i-0eb935deffa4b32fb` |

- **Subnet Range:** `172.31.64.0/20` (Netmask: `255.255.240.0`, Broadcast: `172.31.79.255`)

**Calico IPAM Allocation**

| Scope | CIDR | Assigned To |
|---|---|---|
| Global Pod Cluster CIDR | `192.168.0.0/16` | Cluster-wide |
| Node 1 IPAM Block | `192.168.180.192/26` | worker-node1 |
| Node 2 IPAM Block | `192.168.203.128/26` | worker-node2 |

---

## 3. The Core Problem: Why the Failure Occurred

To understand why this issue surfaced, it is necessary to examine how AWS virtualized networking interacts with Calico's internal routing modes.

### 3.1 What Is AWS EC2 "Source/Destination Checking"?

By default, the AWS Nitro hypervisor enforces strict anti-spoofing validation rules on every Elastic Network Interface (ENI).

```text
[EC2 Instance]
      |
      | Transmits packet: [Src IP: X | Dst IP: Y]
      v
[AWS Hypervisor / Nitro Controller]
      |
      +--> Is Src IP == ENI's assigned private IP?
      |      AND
      +--> Is Dst IP == ENI's assigned private IP (for inbound)?
      |
      +-- YES ---> Forward through VPC substrate
      |
      +-- NO  ---> DROP SILENTLY (blackhole)
```

1. **Egress Check:** When an operating system transmits an Ethernet frame over its physical or virtual interface (e.g., `ens5`), the underlying hypervisor intercepts it. If the **IPv4 source address** embedded in the outer packet header does not match an IP address formally registered to that ENI in the EC2 control plane (either the primary private IPv4 address or an explicitly allocated secondary IPv4 address), **the hypervisor drops the packet immediately**.
2. **Ingress Check:** If a packet arrives at an ENI whose **IPv4 destination address** does not match any IP registered to that ENI in AWS, the hypervisor drops the packet. It assumes the traffic was routed to the wrong hardware address or represents an unauthorized interception.

This security policy prevents compromised instances from performing man-in-the-middle attacks, spoofing IP addresses, or participating in unauthorized routing loops. However, the same check also prevents an EC2 instance from acting as an L3 router or gateway for other networks unless explicitly permitted.

### 3.2 How Calico's CrossSubnet Mode Triggered the Problem

Calico allows administrators to define how Pod-to-Pod traffic is transported across nodes using an `IPPool` custom resource. The pool in this cluster was configured as follows:

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  ipipMode: Never
  vxlanMode: CrossSubnet
  natOutgoing: true
```

The key setting is `vxlanMode: CrossSubnet`. Calico applies this mode via a state-machine optimization:

| Condition | Calico Transport Engine | Packet Structure | Requires AWS Source/Dest Check Disabled? |
|---|---|---|---|
| **Nodes on different subnets** (inter-subnet / across L3 boundaries) | **Encapsulated:** Full VXLAN overlay (UDP port 4789). Outer header contains node IPs. | `[IP: Src=Node1, Dst=Node2] [UDP: 4789] [VXLAN] [Inner IP: Src=Pod1, Dst=Pod2]` | **No.** AWS only evaluates the outer IP header (Node1 → Node2), which matches the ENI addresses. |
| **Nodes on the same subnet** (intra-subnet / same L2-L3 domain) | **Unencapsulated:** Direct L3 host routing. The outer header contains the raw Pod IPs. | `[IP: Src=Pod1, Dst=Pod2] [Payload]` | **Yes.** AWS inspects the raw Pod IPs (`192.168.x.x`), finds they don't match the ENI, and drops the frame. |

Because worker-node1 (`172.31.67.162`) and worker-node2 (`172.31.69.237`) both resided inside the single AWS VPC subnet `172.31.64.0/20`, Calico disabled VXLAN encapsulation for performance reasons — saving 50 bytes of overhead per packet and avoiding UDP encapsulation processing.

Calico configured the nodes to behave as conventional routers: worker-node1 placed an unencapsulated IP packet onto the wire with:

- **Source IP:** `192.168.180.195` (Pod 1)
- **Destination IP:** `192.168.203.131` (Pod 2)
- **Layer 2 destination MAC:** The gateway MAC of the AWS VPC router, or the direct MAC of worker-node2.

As soon as this frame left the driver queue of `ens5` on worker-node1, the AWS Nitro hypervisor inspected the source address. Finding `192.168.180.195` instead of `172.31.67.162`, the hypervisor silently dropped the frame. The packet never reached the AWS physical fabric and never arrived at worker-node2.

---

## 4. Step-by-Step Diagnostic Breakdown

The following sequence reflects the systematic troubleshooting process used to isolate the failure domain.

| Step | Diagnostic Question | Outcome |
|---|---|---|
| 1 | Are both Pods running with assigned IPs? | Yes → proceed |
| 2 | Are all `calico-node` daemons healthy? | Yes → proceed |
| 3 | Can worker-node1 ping worker-node2's IP? | Yes → proceed |
| 4 | Is `net.ipv4.ip_forward=1` and is iptables OK? | Yes → proceed |
| 5 | Does node1 have an L3 route to node2? | Yes → proceed |
| 6 | Do packets leave node1 but fail to reach node2? | Yes — lost in the VPC → proceed |
| 7 | Is EC2 `SourceDestCheck` enabled? | Yes — **root cause found** |
| 8 | Disable AWS EC2 `SourceDestCheck` | Applied |
| 9 | Re-test traffic flow | 0% packet loss |

### Step 1: Pod State & API Verification

First, verify that the workloads are not failing internally due to scheduling constraints, image pull failures, or missing interfaces:

```bash
kubectl get pods -o wide
```

**Output:**

```text
NAME                        READY   STATUS    RESTARTS   AGE   IP                NODE           NOMINATED NODE   READINESS GATES
net-test-79c969fccd-pkjl9   1/1     Running   0          12m   192.168.180.195   worker-node1   <none>           <none>
net-test-79c969fccd-s6rsm   1/1     Running   0          12m   192.168.203.131   worker-node2   <none>           <none>
```

Both workloads were initialized, their network namespaces were plumbed by the CNI, valid IP addresses were assigned from their respective node IPAM blocks, and the local endpoints were operational.

### Step 2: Calico DaemonSet and Node Agent Health

Verify that the CNI control plane and data plane agents are functional:

```bash
kubectl get daemonset calico-node -n calico-system
```

**Output:**

```text
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
calico-node   3         3         3       3            3           kubernetes.io/os=linux   4d
```

All daemon Pods were healthy, with zero restart loops. The BGP/routing agents (`bird`) and configuration managers (`felix`) were actively processing cluster updates.

### Step 3: Underlay Host-to-Host (L3) Verification

Verify that the AWS VPC underlay network permits communication between the hosts themselves over the native VPC IPs.

From worker-node1 (`172.31.67.162`):

```bash
ping -c 3 172.31.69.237
```

**Output:**

```text
PING 172.31.69.237 (172.31.69.237) 56(84) bytes of data.
64 bytes from 172.31.69.237: icmp_seq=1 ttl=64 time=0.182 ms
64 bytes from 172.31.69.237: icmp_seq=2 ttl=64 time=0.191 ms
64 bytes from 172.31.69.237: icmp_seq=3 ttl=64 time=0.178 ms

--- 172.31.69.237 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2048ms
```

The AWS security group, subnets, route tables, and underlying hardware allowed uninterrupted bidirectional communication between the two node interfaces.

### Step 4: Linux Kernel Forwarding & iptables Audit

When a Linux host acts as an intermediary router between a local container interface (e.g., `cali1a2b3c`) and a physical interface (`ens5`), the kernel must be allowed to forward IPv4 packets. Check the system configuration:

```bash
sysctl net.ipv4.ip_forward
# Output: net.ipv4.ip_forward = 1
```

Next, verify that iptables is not silently dropping forwarded packets via its default policies:

```bash
iptables -S FORWARD
```

**Output:**

```text
-P FORWARD ACCEPT
-A FORWARD -m comment --comment "cali:wUH94_8123" -j cali-FORWARD
```

The `FORWARD` chain used a default `ACCEPT` policy. Calico's filter rules (`cali-FORWARD`) were loaded into the netfilter tables, confirming that neither the Linux kernel nor the local firewall dropped the traffic.

### Step 5: Verification of the Kernel Routing Tables

Inspect the Linux routing table on worker-node1 to check whether the host knows how to reach the remote Pod network (`192.168.203.128/26`):

```bash
ip route show
```

**Output on worker-node1:**

```text
default via 172.31.64.1 dev ens5 proto dhcp src 172.31.67.162 metric 100
172.31.64.0/20 dev ens5 proto kernel scope link src 172.31.67.162 metric 100
192.168.180.192/26 dev cali-local proto bird
192.168.180.195 dev cali1a2b3c4d scope link
192.168.203.128/26 via 172.31.69.237 dev ens5 proto 80 onlink
```

**Output on worker-node2:**

```text
default via 172.31.64.1 dev ens5 proto dhcp src 172.31.69.237 metric 100
172.31.64.0/20 dev ens5 proto kernel scope link src 172.31.69.237 metric 100
192.168.180.192/26 via 172.31.67.162 dev ens5 proto 80 onlink
192.168.203.128/26 dev cali-local proto bird
192.168.203.131 dev cali9z8y7x6w scope link
```

> **Key finding — line 5 on worker-node1:**
> ```text
> 192.168.203.128/26 via 172.31.69.237 dev ens5 proto 80 onlink
> ```

- `proto 80` indicates that the route was installed directly by the Calico Felix/BIRD agent.
- The route explicitly instructs the kernel: *to reach any IP in the block `192.168.203.128/26` (which includes the target Pod `192.168.203.131`), do not encapsulate it — transmit it directly out of device `ens5` with a Layer 3 next-hop of `172.31.69.237`.*
- Notice the absence of a `vxlan.calico` egress device. Calico determined that worker-node2 shared the same subnet (`172.31.64.0/20`) and bypassed VXLAN encapsulation entirely.

### Step 6: Packet Capture and Failure Domain Isolation

To determine whether packets were dropping inside the Linux host or downstream within AWS, run `tcpdump` on both ends during an active connectivity test.

**Terminal 1 (ingress capture on destination node: worker-node2):**

```bash
tcpdump -nnvv -i ens5 host 192.168.180.195 or host 192.168.203.131
```

**Terminal 2 (egress capture on source node: worker-node1):**

```bash
tcpdump -nnvv -i ens5 host 192.168.180.195 or host 192.168.203.131
```

**Terminal 3 (run the ping test from inside the source Pod):**

```bash
kubectl exec -it net-test-79c969fccd-pkjl9 -- ping -c 2 192.168.203.131
```

**Results:**

*Egress on worker-node1 (`ens5`):*

```text
14:22:01.102381 IP (tos 0x0, ttl 63, id 41231, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.180.195 > 192.168.203.131: ICMP echo request, id 14, seq 1, length 64
14:22:02.103412 IP (tos 0x0, ttl 63, id 41232, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.180.195 > 192.168.203.131: ICMP echo request, id 14, seq 2, length 64
```

Result: Packets passed cleanly through iptables, hit the kernel routing table, and were transmitted out of the physical interface `ens5`.

*Ingress on worker-node2 (`ens5`):*

```text
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

Result: **Zero packets arrived at the destination network interface.**

**Conclusion:** The packet was transmitted by the source operating system onto the virtual wire but never reached the destination operating system. Because both nodes were hosted in the same VPC subnet, the drop occurred within the intermediate network layer — the **AWS Nitro hypervisor / VPC virtual switch**.

### Step 7: Identifying the AWS Configuration Defect

Query the AWS API using the AWS CLI to check the network configuration of the EC2 instances:

```bash
# Check worker-node1
aws ec2 describe-instance-attribute \
    --instance-id i-0281a0d1ab953427e \
    --attribute sourceDestCheck \
    --query '{InstanceId:InstanceId, SourceDestCheck:SourceDestCheck.Value}'

# Check worker-node2
aws ec2 describe-instance-attribute \
    --instance-id i-0eb935deffa4b32fb \
    --attribute sourceDestCheck \
    --query '{InstanceId:InstanceId, SourceDestCheck:SourceDestCheck.Value}'
```

**Output:**

```json
{
    "InstanceId": "i-0281a0d1ab953427e",
    "SourceDestCheck": true
}
{
    "InstanceId": "i-0eb935deffa4b32fb",
    "SourceDestCheck": true
}
```

Both instances had `SourceDestCheck: true`.

When worker-node1 transmitted a packet with source IP `192.168.180.195` (a Pod IP, not matching the ENI's AWS-assigned IP `172.31.67.162`), the AWS hypervisor flagged it as an address-spoofing attempt and dropped it. Even if the packet had made it across, worker-node2 would have rejected it on receipt, since the destination address (`192.168.203.131`) did not match that node's ENI IP (`172.31.69.237`).

---

## 5. Remediation Plan and Validation

### Step 1: Immediate Remediation via AWS CLI

Disable source/destination validation on both EC2 worker instances.

```bash
# Disable on worker-node1
aws ec2 modify-instance-attribute \
    --instance-id i-0281a0d1ab953427e \
    --no-source-dest-check

# Disable on worker-node2
aws ec2 modify-instance-attribute \
    --instance-id i-0eb935deffa4b32fb \
    --no-source-dest-check
```

Confirm the attribute change:

```bash
aws ec2 describe-instances \
    --instance-ids i-0281a0d1ab953427e i-0eb935deffa4b32fb \
    --query "Reservations[].Instances[].{ID:InstanceId,SourceDestCheck:SourceDestCheck}"
```

**Output:**

```json
[
    {
        "ID": "i-0281a0d1ab953427e",
        "SourceDestCheck": false
    },
    {
        "ID": "i-0eb935deffa4b32fb",
        "SourceDestCheck": false
    }
]
```

### Step 2: Post-Fix Validation

#### Test 1: Node-to-Remote Pod (Direct Host Routing Validation)

Run a ping from the host network namespace of worker-node1 directly to the remote Pod running on worker-node2:

```bash
ping -c 3 192.168.203.131
```

**Output:**

```text
PING 192.168.203.131 (192.168.203.131) 56(84) bytes of data.
64 bytes from 192.168.203.131: icmp_seq=1 ttl=63 time=0.341 ms
64 bytes from 192.168.203.131: icmp_seq=2 ttl=63 time=0.320 ms
64 bytes from 192.168.203.131: icmp_seq=3 ttl=63 time=0.312 ms

--- 192.168.203.131 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2031ms
```

#### Test 2: In-Cluster End-to-End Pod-to-Pod Communication

Execute an ICMP ping session between the two test Pods across the network boundary:

```bash
kubectl exec -it net-test-79c969fccd-pkjl9 -- ping -c 4 192.168.203.131
```

**Output:**

```text
PING 192.168.203.131 (192.168.203.131): 56 data bytes
64 bytes from 192.168.203.131: seq=0 ttl=62 time=0.418 ms
64 bytes from 192.168.203.131: seq=1 ttl=62 time=0.285 ms
64 bytes from 192.168.203.131: seq=2 ttl=62 time=0.308 ms
64 bytes from 192.168.203.131: seq=3 ttl=62 time=0.337 ms

--- 192.168.203.131 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.285/0.337/0.418 ms
```

The packets now traverse the AWS VPC layer without being dropped. Notice that the packet reports `ttl=62` on arrival:

| TTL Observed | Explanation |
|---|---|
| 64 | Generated by the container network stack |
| 63 | Decremented by worker-node1 when routing from `cali1a2b3c4d` out to `ens5` |
| 62 | Decremented by worker-node2 when routing from `ens5` into the target container's interface, `cali9z8y7x6w` |

---

## 6. Comparing Architectural Approaches

Disabling `SourceDestCheck` resolved the issue for Calico's direct routing mode. However, there are multiple ways to configure Kubernetes networking on AWS. The table below compares the architectural trade-offs between them.

| Metric / Consideration | Calico Direct Routing (CrossSubnet, same subnet) | Calico Full Overlay (`vxlanMode: Always` / `ipipMode: Always`) | AWS VPC CNI (`amazon-vpc-cni-k8s`) |
|---|---|---|---|
| **AWS Source/Dest Check** | **Must be disabled** (`false`). AWS must allow routing of foreign Pod IPs. | **Can remain enabled** (`true`). Traffic is encapsulated within outer node IP headers. | **Can remain enabled** (`true`). Every Pod receives an officially assigned secondary AWS VPC IP. |
| **Encapsulation Overhead** | **0 bytes.** Packets run natively at the instance MTU (typically 9001 bytes with AWS jumbo frames). | **50 bytes** (VXLAN: 14 Ethernet + 20 IP + 8 UDP + 8 VXLAN) or **20 bytes** (IPIP). | **0 bytes.** Native VPC routing; no overlay network is used. |
| **CPU Performance** | **Optimal.** No encapsulation or decapsulation overhead in the Linux network stack. | **Slight CPU overhead** from packet encapsulation/decapsulation under high network throughput. | **Optimal.** Line-rate performance using hardware-native ENI attachments. |
| **VPC IP Address Consumption** | **Minimal.** Only the EC2 worker instances consume VPC IP addresses. | **Minimal.** Pod CIDRs are decoupled from the AWS VPC subnet space. | **High.** Every Pod consumes a real private IPv4 address from the AWS VPC subnet. |
| **Cross-VPC / Peering Complexity** | **Requires BGP / AWS route tables.** VPC routes must be aware of Pod CIDRs to cross VPC boundaries. | **Zero VPC awareness needed.** Overlays route transparently across peered VPCs or Direct Connect. | **Seamless.** Fully routable throughout the AWS VPC, Transit Gateway, and on-premises footprint. |
| **Kubernetes Scale Limits** | Bound by Calico IPAM and the underlying Linux routing table size. | Scales to large cluster sizes; decoupled from cloud provider network limits. | Limited by the maximum number of ENIs and secondary private IPs supported by the EC2 instance type. |

---

## 7. Operationalizing the Fix (Infrastructure as Code)

To prevent this issue from recurring when instances are replaced, terminated, or scaled out, the `SourceDestCheck` modification must be embedded in infrastructure automation workflows.

### 7.1 HashiCorp Terraform

If instances are managed using standalone `aws_instance` resources:

```hcl
resource "aws_instance" "k8s_worker" {
  count         = 2
  ami           = "ami-04b70fa74e45c3917" # Ubuntu 24.04 LTS
  instance_type = "t3.medium"
  subnet_id     = "subnet-0123456789abcdef0"

  # CRITICAL: Must be explicitly disabled for Calico direct routing
  source_dest_check = false

  vpc_security_group_ids = [aws_security_group.k8s_nodes.id]

  tags = {
    Name = "k8s-worker-${count.index + 1}"
    Role = "kubernetes-worker"
  }
}
```

If worker nodes are managed via an **Auto Scaling Group (ASG)** with an **AWS Launch Template**:

```hcl
resource "aws_launch_template" "k8s_worker_template" {
  name_prefix   = "k8s-worker-template-"
  image_id      = "ami-04b70fa74e45c3917"
  instance_type = "t3.medium"

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.k8s_nodes.id]

    # CRITICAL: Disable Source/Destination checking on the network interface
    source_dest_check = false
  }

  user_data = filebase64("${path.module}/userdata.sh")

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "k8s-worker-node"
    }
  }
}
```

### 7.2 AWS CloudFormation

In a CloudFormation template provisioning an EC2 instance:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Kubernetes Worker Instance Template
Resources:
  K8sWorkerNode1:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.medium
      ImageId: ami-04b70fa74e45c3917
      SubnetId: subnet-0123456789abcdef0
      # CRITICAL: Disable SourceDestCheck
      SourceDestCheck: false
      SecurityGroupIds:
        - sg-0123456789abcdef0
      Tags:
        - Key: Name
          Value: worker-node1
```

### 7.3 Automated Audit via AWS CLI

To quickly audit an entire cluster and ensure all instances matching a tag have `SourceDestCheck` disabled:

```bash
# Find all instances tagged with Role=kubernetes-worker that have SourceDestCheck enabled
aws ec2 describe-instances \
    --filters "Name=tag:Role,Values=kubernetes-worker" "Name=instance-state-name,Values=running" \
    --query "Reservations[].Instances[?SourceDestCheck==\`true\`].InstanceId" \
    --output text | tr '\t' '\n' | while read -r INSTANCE_ID; do
        if [ -n "$INSTANCE_ID" ]; then
            echo "Fixing SourceDestCheck on: ${INSTANCE_ID}"
            aws ec2 modify-instance-attribute --instance-id "${INSTANCE_ID}" --no-source-dest-check
        fi
    done
```

---

## 8. Alternative Fix: Switching Calico to Full VXLAN Encapsulation

If security policy or organizational compliance mandates that `SourceDestCheck` must remain enabled (`true`) across all AWS EC2 instances, the cluster's network configuration must instead be adjusted so that packets are encapsulated before leaving the host.

This can be done by changing the Calico `IPPool` configuration to use `vxlanMode: Always`:

```bash
# Export the current IPPool manifest
kubectl get ippool default-ipv4-ippool -o yaml > ippool.yaml
```

Modify the YAML file to change `vxlanMode` from `CrossSubnet` to `Always`:

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 192.168.0.0/16
  ipipMode: Never
  natOutgoing: true
  vxlanMode: Always # Changed from CrossSubnet to Always
```

Apply the updated manifest:

```bash
kubectl apply -f ippool.yaml
```

After applying the change, Calico's node agents update the host routing table. Check the routes on worker-node1:

```bash
ip route show
```

**Expected updated route:**

```text
192.168.203.128/26 via 192.168.203.128 dev vxlan.calico onlink
```

**Traffic flow with `vxlanMode: Always`:**

1. The packet leaves the Pod and hits the `vxlan.calico` virtual network interface on worker-node1.
2. The Linux kernel encapsulates the entire Pod IP packet into an outer UDP frame (port 4789).
3. The outer IP header has a source of `172.31.67.162` and a destination of `172.31.69.237` (the nodes' EC2 addresses).
4. The AWS Nitro hypervisor checks the outer source and destination IPs. Since they match the node's ENI IP, the packet passes `SourceDestCheck` verification.
5. Upon arrival at worker-node2, the kernel decapsulates the UDP packet and delivers the inner frame (`192.168.180.195` → `192.168.203.131`) to the destination Pod.

> **Note:** When using full VXLAN overlay mode, ensure that the network interface MTU accounts for the 50 bytes of encapsulation overhead — e.g., set Calico's MTU to `8951` if using AWS jumbo frames (9001-byte MTU), or `1450` if using a standard 1500-byte MTU.

---

## 9. Comprehensive Health-Check Script

To simplify cluster validation, run the following automated verification script on an administrative workstation with `kubectl` and `aws-cli` installed:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=========================================================="
echo "    KUBERNETES ON AWS NETWORKING HEALTH AUDIT SCRIPT      "
echo "=========================================================="

echo -e "\n[1/5] Checking Kubernetes Nodes State..."
kubectl get nodes -o wide

echo -e "\n[2/5] Inspecting Calico IPPool Configuration..."
if kubectl get crd ippools.projectcalico.org >/dev/null 2>&1; then
    kubectl get ippools.projectcalico.org -o custom-columns=\
NAME:.metadata.name,CIDR:.spec.cidr,IPIP_MODE:.spec.ipipMode,VXLAN_MODE:.spec.vxlanMode
else
    echo "Warning: Calico ProjectCalico CRD not detected via kubectl."
fi

echo -e "\n[3/5] Verifying Linux Kernel IP Forwarding on Nodes..."
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
    echo -n "Node: ${node} -> "
    # If running over SSH or SSM:
    # ssh ubuntu@${node} "sysctl net.ipv4.ip_forward"
    echo "Requires sysctl net.ipv4.ip_forward == 1"
done

echo -e "\n[4/5] Auditing AWS EC2 SourceDestCheck Status..."
NODE_IPS=$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}')

for ip in ${NODE_IPS}; do
    INSTANCE_INFO=$(aws ec2 describe-instances \
        --filters "Name=private-ip-address,Values=${ip}" \
        --query "Reservations[].Instances[][InstanceId,SourceDestCheck]" \
        --output text)

    INST_ID=$(echo "${INSTANCE_INFO}" | awk '{print $1}')
    SRC_CHK=$(echo "${INSTANCE_INFO}" | awk '{print $2}')

    if [ "${SRC_CHK}" == "True" ] || [ "${SRC_CHK}" == "true" ]; then
        echo "[-] WARNING: Node IP ${ip} (Instance: ${INST_ID}) has SourceDestCheck = ENABLED."
        echo "    If running unencapsulated CNI routes, cross-node traffic will be dropped by AWS!"
    else
        echo "[+] Node IP ${ip} (Instance: ${INST_ID}) has SourceDestCheck = DISABLED (Correct for routing)."
    fi
done

echo -e "\n[5/5] Testing Cross-Node Pod Ping Connectivity..."
POD_NODES=$(kubectl get pods -l app=net-test -o jsonpath='{range .items[*]}{.metadata.name}{","}{.status.podIP}{","}{.spec.nodeName}{"\n"}{end}')

if [ $(echo "${POD_NODES}" | wc -l) -ge 2 ]; then
    P1_NAME=$(echo "${POD_NODES}" | sed -n '1p' | cut -d',' -f1)
    P2_IP=$(echo "${POD_NODES}" | sed -n '2p' | cut -d',' -f2)
    P2_NODE=$(echo "${POD_NODES}" | sed -n '2p' | cut -d',' -f3)

    echo "Attempting ping from Pod: ${P1_NAME} to target IP: ${P2_IP} on Node: ${P2_NODE}..."
    if kubectl exec "${P1_NAME}" -- ping -c 3 -W 2 "${P2_IP}"; then
        echo -e "\n[+] SUCCESS: Cross-node Pod communication is operational!"
    else
        echo -e "\n[-] ERROR: Cross-node Pod communication failed. Check AWS SourceDestCheck and security groups."
    fi
else
    echo "Notice: Deploy a multi-replica test deployment (e.g. app=net-test) across different nodes to run the live test."
fi

echo -e "\nAudit complete."
```

---

## 10. Summary Checklist for Post-Incident Verification

1. **Verify Pod Readiness:** Confirm that Pods are assigned IPs from distinct node CIDRs and are in the `Running` state without crash loops (`kubectl get pods -o wide`).
2. **Review Calico IPAM:** Run `kubectl get ippools -o yaml` to identify whether encapsulation is set to `Always`, `CrossSubnet`, or `Never`.
3. **Inspect the Local Routing Table:** Confirm that non-local Pod CIDRs have explicit routes pointing to the remote worker nodes (`ip route show`).
4. **Audit AWS Instance Attributes:** For unencapsulated traffic routing, confirm that every worker node's ENI reports `"SourceDestCheck": false` in the AWS API.
5. **Verify Security Groups:** Ensure that worker-to-worker security group rules allow traffic for:
   - **Direct Routing:** All traffic, or the internal Pod CIDR (`192.168.0.0/16`), across node interfaces.
   - **VXLAN Routing:** UDP port `4789` between all node private IPs.
   - **BGP Routing:** TCP port `179` between all node private IPs (if Calico BGP peering is used).
6. **Codify the Configuration:** Add `source_dest_check = false` to your Terraform or CloudFormation templates to ensure that auto-scaled or replaced instances retain the required networking settings.
