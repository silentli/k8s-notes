# SOP: Manual Kubernetes Installation on VMs - 02. Control Plane Setup

## Objective
This procedure covers the installation of Kubernetes software binaries, initialization of the cluster, configuration of administrative credentials, and deployment of the eBPF networking layer (Cilium) on the Control Plane node.

*Note: Ensure the steps from **01. Common Prerequisites and OS Setup** have been fully completed on the target machine prior to this.*

## 1. Kubernetes Binaries Installation
Install the required cluster management tools on the Control Plane node.
1. Download the public signing key for the Kubernetes package repositories:
   ```bash
   sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg --yes
   ```
2. Add the appropriate Kubernetes `apt` repository:
   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
3. Because this is the Master Node, include `kubectl` in the installation:
   ```bash
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   ```
4. Lock the packages to prevent automatic updates:
   ```bash
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

## 2. Cluster Initialization (`kubeadm init`)
Initialize the primary API server, start the `etcd` database, and configure the Certificate Authority.
1. Run `kubeadm init`, specifying the IP address the API server will listen on, and what internal IP range the pod network will use:
   ```bash
   sudo kubeadm init --apiserver-advertise-address=<ip_address> --pod-network-cidr=10.244.0.0/16
   ```

## 3. Administrative Credentials (`kubeconfig`)
Establish permissions for the standard administrator user to interact with the API via `kubectl`.
1. Copy the newly generated admin configuration file into the current user's home directory:
   ```bash
   mkdir -p \$HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf \$HOME/.kube/config
   sudo chown \$(id -u):\$(id -g) \$HOME/.kube/config
   ```
2. Verify the API server responds and shows the Master Node:
   ```bash
   kubectl get nodes
   ```

## 4. Verify Network Ports
Ensure the `kube-apiserver` binds correctly and actively listens to network requests.
1. Audit the internal state to prove the software is running and listening:
   ```bash
   sudo ss -tulpn
   # or
   sudo ss -tulpn | grep 6443
   ```

## 5. Installing Helm
Deploy Helm on the Control Plane node to deploy complex software architectures.
1. Download the cryptographic signing key:
   ```bash
   curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
   ```
2. Install HTTPS transport and add the repository:
   ```bash
   echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
   ```
3. Installation:
   ```bash
   sudo apt-get update
   sudo apt-get install -y helm
   ```
4. Validation:
   ```bash
   helm version
   ```

## 6. Install Cilium as CNI
Install Cilium to handle pod-to-pod networking overlay using eBPF technology.
1. Add the Cilium official Helm repository:
   ```bash
   helm repo add cilium https://helm.cilium.io/
   ```
2. Force Helm to update its local cache:
   ```bash
   helm repo update
   ```
3. Execute the deployment command:
   ```bash
   helm install cilium cilium/cilium --namespace kube-system
   ```
4. Verify the Cilium pods are running:
   ```bash
   kubectl get pods -n kube-system -l k8s-app=cilium
   ```
5. Verify the node network change state transition (should be `Ready`):
   ```bash
   kubectl get nodes
   ```
