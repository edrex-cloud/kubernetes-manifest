# ON-PREMISES K8s CLUSTER DEPLOYMENT (kubeadm + Cilium)

---

A manual, production-style 3-node Kubernetes cluster deployed on AWS EC2 instances using kubeadm, containerd, and Cilium CNI. No managed services, no EKS, no shortcuts. Every component configured by hand.

---

## ARCHITECTURE

```text
┌─────────────────────────────────────────────────────┐
│                   AWS EC2 (Ubuntu 24.04)            │
│                                                     │
│  ┌─────────────────────┐                            │
│  │ master-node-cluster │  Control Plane             │
│  │ (kube-apiserver,    │  Private IP: X.X.X.X       │
│  │  etcd, scheduler,   │  Public IP:  X.X.X.X       │
│  │  controller-manager)│                            │
│  └──────────┬──────────┘                            │
│             │ port 6443                             │
│       ┌─────┴────────┐                              │
│       │              │                              │
│  ┌────┴───────┐  ┌───┴────────┐                     │
│  │worker-node-1│ │worker-node-2│                    │
│  │ (kubelet,  │  │ (kubelet,  │                     │
│  │ kube-proxy,│  │ kube-proxy,│                     │
│  │ cilium)    │  │ cilium)    │                     │
│  └────────────┘  └────────────┘                     │
└─────────────────────────────────────────────────────┘

CNI: Cilium v1.19.3
Container Runtime: containerd (Docker Engine)
Kubernetes Version: v1.34.9
Pod Network CIDR: 192.168.0.0/16
```


## Stack

* **Cloud:** AWS EC2 (Ubuntu 24.04 LTS)
* **Kubernetes:** v1.34.9 via kubeadm
* **Container Runtime:** containerd (bundled with Docker Engine)
* **CNI:** Cilium v1.19.3
* **Cluster Management UI:** Freelens (OpenLens successor)

---


## Prerequisites

* **3 AWS EC2 instances** (Ubuntu 24.04), minimum t2.medium for master
* **Security group inbound rules:**
  * Port 22 (SSH) from your IP
  * Port 6443 (Kubernetes API server) between all nodes
  * Port 10250 (kubelet API) between all nodes
  * All traffic allowed between nodes within the VPC CIDR
* **Each instance assigned a unique hostname** (master-node-cluster, worker-node-1, worker-node-2)

---

## Setup: All Nodes (Master + Workers)

Run all steps in this section on every node before proceeding to master-only steps.

### 1. Update the system

```bash
sudo apt update && sudo apt upgrade -y
```
### 2. Disable Swap
Kubernetes requires swap to be disabled on all nodes

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
### 3. Enable kernel modules and sysctl settings

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```
### 4. Install Docker and containerd

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker ubuntu
sudo systemctl start docker
sudo systemctl enable docker
```
### 5. Configure containerd for Kubernetes
Docker's default containerd config is not kubelet-compatible. Reconfigure it:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Switch cgroup driver to systemd (required to match kubelet)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Pin pause container image version
sudo sed -i 's|sandbox = "registry.k8s.io/pause:.*"|sandbox = "registry.k8s.io/pause:3.10"|' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```
### 6.  Add Kubernetes APT repository and install kubeadm, kubelet, kubectl

```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
---

## Setup for Master Node Only

### 1. Initialize the control plane
 ```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
> 💡 **Note:** Choose a pod CIDR that does not overlap with your VPC CIDR. This cluster used `192.168.0.0/16` to avoid conflict with the VPC's `10.0.0.0/16` range.

### 2. Set up Kubeconfig
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### 3. Install Cilium CLI
```bash
curl -L --remote-name https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
sudo tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin
rm cilium-linux-amd64.tar.gz
```
### 4. Install Cilium CNI
```bash
cilium install
```
Wait 5 minutes for Cilium pods to initialize, then verify:
```bash
kubectl get nodes
cilium status
```
---

## Worker Nodes Cluster Connection

After the master node is initialized, run the join command printed at the end of kubeadm init on each worker node:
```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> \
        --discovery-token-ca-cert-hash sha256:<HASH>
```
>  💡 **The join token expires after 24 hours. To regenerate: `kubeadm token create --print-join-command` on the master.**

## Verify Cluster Health
```bash
kubectl get nodes -o wide
kubectl get pods -A
cilium status
kubectl cluster-info
```
## Expected output: all 3 nodes in Ready state, all kube-system pods Running. ✅

---

## TROUBLESHOOTING
### Issue 1: Worker node cannot reach API server (port 6443 timeout)
Symptom:
```bash
error: couldn't validate the identity of the API Server: context deadline exceeded
```
Cause: Security group on the master node was not allowing inbound TCP port 6443 from worker nodes.

Fix: Add inbound rule to the master's security group allowing TCP 6443 from the VPC CIDR (or worker node IPs). Retry the `kubeadm join` command

### Issue 2: Apiserver TLS certificate does not include public IP (Openlens/remote kubectl connection fails)
Symptom: Connecting to the cluster from a remote machine (Freelens/kubectl) fails with a TLS certificate error because the cert was only signed for the private IP, not the public IP.

Fix: Regenerate the apiserver certificate with the public IP as a SAN

Step 1: Export the current cluster config:
```bash
kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm-config.yaml
```
Step 2: Edit kubeadm-config.yaml and add the public IP under certSANs:
```bash
apiServer:
  certSANs:
  - "172.31.22.194"
  - "YOUR_PUBLIC_IP"
```
Step 3: Back up and remove the old cert:
```bash
sudo mv /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.old
sudo mv /etc/kubernetes/pki/apiserver.key /etc/kubernetes/pki/apiserver.key.old
```
Step 4: Generate new cert:
```bash
sudo kubeadm init phase certs apiserver --config kubeadm-config.yaml
```
Step 5: Regenerate kubeconfig:
```bash
rm -rf $HOME/.kube/config
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Step 6: Copy the contents of `cat $HOME/.kube/config` into Openlens when adding the cluster.

### Issue 3: All nodes disappear after accidental deletion via Openlens
Symptom: kubectl get nodes returns No resources found after nodes were deleted from the Freelens UI. Kubelet logs show repeated "Error updating node status" errors.

Cause: Deleting a node object in Freelens removes the Node resource from etcd. Kubelet keeps running but cannot re-register the node on its own without a restart.

Fix: Restart kubelet on each affected node, starting with the master:
```bash
sudo systemctl restart kubelet
sleep 15
kubectl get nodes
```

Repeat on each worker. Nodes self-register within 15 to 30 seconds. After recovery, restore the control-plane role label on the master:
```bash
kubectl label node master-node-cluster node-role.kubernetes.io/control-plane=
```
> 💡 **Lesson learned:** Openlens node deletion is destructive. Use it for observation, not for managing node lifecycle.
---
## Key Learnings

* containerd requires explicit systemd cgroup configuration to work with kubelet. Docker's bundled containerd ships with the wrong default.
* kubeadm-generated TLS certs only include IPs known at init time. Any remote access via a public IP requires cert regeneration with updated SANs.
* Deleting a Kubernetes Node object does not stop the kubelet process on that node. A kubelet restart triggers re-registration without needing to re-run kubeadm join.
* Pod CIDR selection matters. Overlapping with your VPC or node subnet CIDR will silently break pod networking.
* Cilium requires port 10250 open between nodes for full health reporting. Security group restrictions on kubelet's API port cause partial connectivity errors in cilium status even when the CNI is otherwise functional.
---
## Connect with me

* 💼 **LinkedIn:** [Divine Eric](https://www.linkedin.com/in/divine-eric-a06733373/)
* 🐙 **GitHub:** [edrex-cloud](https://github.com/edrex-cloud)



