# Kubernetes Multi-Node Cluster from Scratch (kubeadm)

A three-node Kubernetes cluster built manually on AWS EC2 using kubeadm, with one master (control plane) and two worker nodes. This project focuses on understanding Kubernetes cluster construction at a fundamental level, rather than relying on managed services or lightweight distributions.

---

## Why Build a Cluster Manually

Following a fundamentals-first learning approach, this build uses kubeadm to construct every part of the cluster by hand: the container runtime, the Kubernetes components, the control plane initialization, the pod network, and the worker join process. This is deliberately more involved than a single-node K3s setup or a managed EKS cluster. The goal is to understand what those higher-level tools automate away, which makes troubleshooting and operating any Kubernetes environment far easier.

---

## Architecture

```
                    ┌────────────────────────────┐
                    │   k8s-master (t2.medium)    │
                    │   Control plane:            │
                    │   API server, scheduler,    │
                    │   etcd, controller manager  │
                    └──────────────┬─────────────┘
                                   │ (private network, port 6443)
                 ┌─────────────────┴─────────────────┐
                 │                                   │
    ┌────────────────────────┐         ┌────────────────────────┐
    │  k8s-worker-1           │         │  k8s-worker-2           │
    │  Runs pods              │         │  Runs pods              │
    │  Pod network 10.244.1.x │         │  Pod network 10.244.2.x │
    └────────────────────────┘         └────────────────────────┘
```

---

## Cluster Specifications

| Node | Instance Type | Role |
|---|---|---|
| k8s-master | t2.medium (2 vCPU required by kubeadm) | Control plane |
| k8s-worker-1 | t2.micro | Worker |
| k8s-worker-2 | t2.micro | Worker |

| Component | Version / Choice |
|---|---|
| OS | Ubuntu 24.04 |
| Container runtime | containerd (systemd cgroup driver) |
| Kubernetes | v1.31 (pinned, identical on all nodes) |
| Pod network (CNI) | Flannel (10.244.0.0/16) |

---

## Security Group Configuration

All three nodes share one security group. The key rule is a self-referencing entry that allows nodes to communicate freely on the internal network.

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | My IP | Access |
| All traffic | All | The security group's own ID | Node-to-node communication |

The self-referencing rule (source = the security group itself) is what allows the master and workers to reach each other across Kubernetes' many internal ports without enumerating each one. This is the single most common cause of worker join failures when missing.

---

## Part 1: Common Setup (All Three Nodes)

These steps run identically on every node.

### Set Hostname

```bash
sudo hostnamectl set-hostname k8s-master      # on master
sudo hostnamectl set-hostname k8s-worker-1    # on worker 1
sudo hostnamectl set-hostname k8s-worker-2    # on worker 2
```

### Disable Swap
Kubernetes requires swap to be off.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Kernel Modules and Networking

```bash
cat <<EOT | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOT

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOT | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOT

sudo sysctl --system
```

### Install containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd conntrack
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

The SystemdCgroup setting must be true so containerd's cgroup driver matches kubelet's. A mismatch causes node instability.

### Install Kubernetes Components

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Pinning a specific version (v1.31) and using apt-mark hold ensures all nodes stay on identical versions, which kubeadm requires for a stable cluster.

---

## Part 2: Initialize the Master (Master Only)

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

The pod-network-cidr value matches what Flannel expects. On success, the output ends with a kubeadm join command containing a token and CA cert hash. This command must be saved, it is what joins the workers.

### Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

At this point the master shows NotReady because there is no pod network yet.

### Install the Pod Network (Flannel)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

After about a minute the master flips to Ready.

---

## Part 3: Join the Workers (Each Worker)

Run the saved join command (with sudo) on each worker:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> \
    --discovery-token-ca-cert-hash sha256:<HASH>
```

Note the master is referenced by its private IP, since workers reach it over the internal network.

---

## Part 4: Verify the Cluster

From the master:

```bash
kubectl get nodes
```

All three nodes should report Ready on the same version. Workers may briefly show NotReady while Flannel configures their networking.

---

## Part 5: Prove Pod Scheduling Across Workers

Deploy a simple test app with multiple replicas and observe how the scheduler distributes them:

```bash
kubectl create deployment nginx-test --image=nginx --replicas=4
kubectl get pods -o wide
```

The -o wide flag shows a NODE column. With 4 replicas across 2 workers, the pods distribute evenly (two per worker). Each worker also receives its own pod-network subnet (10.244.1.x and 10.244.2.x), confirming Flannel is working. This distribution is the entire purpose of a multi-node cluster.

---

## Screenshots

<img width="747" height="182" alt="kube multinode" src="https://github.com/user-attachments/assets/954d6ca6-e99e-4453-a6a8-b431e0b73dba" />

---

## Challenges and Key Learnings

### Master Requires At Least 2 CPUs
kubeadm init fails its preflight check on a 1-CPU instance with "the number of available CPUs 1 is less than the required 2". The instances had launched as t2.micro instead of t2.medium. The fix was to stop the master, change its instance type to t2.medium, and start it again. Forcing past the check with --ignore-preflight-errors was avoided, since a starved control plane is unstable. Workers, by contrast, run fine on t2.micro for light workloads.

### Missing conntrack Package
A second preflight error, "conntrack not found in system path", was resolved by installing the conntrack package on all nodes.

### Resizing Preserves Setup, Changes Public IP
Stopping and resizing an instance keeps its disk (so all prior setup survives) but releases its public IP, assigning a new one on restart. Terminating, by contrast, wipes everything. Knowing this distinction avoided redoing the full node setup.

### Version Consistency Matters
All nodes must run the same Kubernetes version. Pinning v1.31 and holding the packages prevents accidental drift that would destabilize the cluster.

### Self-Referencing Security Group Rule
Worker nodes can only join if they can reach the master across Kubernetes' internal ports. Allowing all traffic from the security group to itself is the simplest way to enable this for a learning cluster.

---

## Cost Management

This is a multi-instance setup (one t2.medium plus two t2.micro running simultaneously), making it the most expensive configuration in this series. A strict spin-up, prove, capture, tear-down discipline keeps costs minimal. All instances were terminated after capturing evidence.

---

## How This Compares

| Setup | Effort | What it teaches |
|---|---|---|
| K3s single-node | One command | Core Kubernetes objects (pods, services, secrets) |
| kubeadm multi-node (this) | Manual, multi-step | How a cluster is actually constructed and how nodes join |
| Managed EKS | Provider-managed control plane | Production operation without the maintenance burden |

---

## Next Steps

- Provision a cluster with Terraform instead of manual steps.
- Move to managed EKS for a production-style control plane.
- Add an Ingress controller in place of NodePort.
- Explore GitOps for declarative, Git-driven deployments.
