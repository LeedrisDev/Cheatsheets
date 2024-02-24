# Kubernetes

## Kubernetes nodes

### Master node

These node plays a crucial role in managing the control API calls for various components within the Kubernetes cluster.
This includes overseeing pods, replication controllers, services, nodes, and more.

### Worker node

Worker nodes are responsible for providing runtime environments for containers. It’s worth noting that a group of 
container pods can extend across multiple worker nodes, ensuring optimal resource allocation and management.

## Pre-requisites

Before diving into the installation, ensure that your environment meets the following prerequisites :

- An Ubuntu 22.04 system.
- Privileged access to the system (root or sudo user).
- Active internet connection.
- Minimum 2GB RAM or more.
- Minimum 2 CPU cores (or 2 vCPUs).
- 20 GB of free disk space on /var (or more).

## Installation

### Step 1: Update the system (all nodes)

Before installing Kubernetes, it’s essential to update the system to the latest packages. To do this, run the following 
commands:

```bash
sudo apt update
sudo apt upgrade
```

### Step 2: Disable swap memory (all nodes)

To enhance Kubernetes performance, disable swap. Run the following commands on all nodes to disable all swaps:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 3: Add Kernel Parameters (all nodes)

Load the required kernel modules on all nodes:

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure the critical kernel parameters for Kubernetes using the following:

```bash
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the changes:

```bash
sudo sysctl --system
```

### Step 4: Install Containerd Runtime (all nodes)

We are using the containerd runtime. Install containerd and its dependencies with the following commands:

```bash
sudo apt install -y curl \
                    gnupg2 \
                    software-properties-common \
                    apt-transport-https \
                    ca-certificates
```

Enable the Docker repository:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Update the package list and install containerd:

```bash
sudo apt update
sudo apt install -y containerd.io
```

Configure containerd to start using systemd as cgroup:

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

Restart and enable the containerd service:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 5: Add Apt Repository for Kubernetes (all nodes)

Kubernetes packages are not available in the default Ubuntu 22.04 repositories. Add the Kubernetes repositories with 
the following commands:

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/kubernetes-xenial.gpg
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

### Step 6: Install Kubectl, Kubeadm, and Kubelet (all nodes)

After adding the repositories, install essential Kubernetes components, including kubectl, kubelet, and kubeadm, on 
all nodes with the following commands:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 7: Initialize Kubernetes Master Node

With all the prerequisites in place, initialize the Kubernetes cluster on the master node using the following Kubeadm 
command:

```bash
sudo kubeadm init
```

After the initialization is complete make a note of the `kubeadm join` command for future reference.

Run the following commands on the master node:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 8: Add Worker Nodes to the Cluster (worker nodes)

On each worker node, use the `kubeadm join` command you noted down earlier:

### Step :9 Install Kubernetes Network Plugin (master node)

Install the Calico network plugin to enable pod networking within the Kubernetes cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```

### Step 10: Verify the cluster and test (master node)

To verify the cluster, run the following command:

```bash
kubectl get pods -n kube-system
kubectl get nodes
```