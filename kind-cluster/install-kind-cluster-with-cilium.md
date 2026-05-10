# Setting Up a kind Cluster with Cilium

**Reference:** [Cilium Official kind Installation Guide](https://docs.cilium.io/en/latest/installation/kind/)

> [!IMPORTANT]
> **Architectural Principle: Separation of Concerns**
> In a production Kubernetes environment, physical worker nodes are highly secure, immutable environments without direct SSH access or standard management tools. `kind` mimics this strict architecture. Tools like `kubectl` and `cilium-cli` should be installed on the **Host OS**, not inside the `kind` container.
> 
> Installing management tools within the `kind` container links their lifecycle to the cluster. Executing `kind delete cluster` would instantly destroy the tools, configurations, and command history. Maintaining these tools on the Host OS ensures a persistent and safe management environment.

## Prerequisites

The following tools must be installed on the host machine prior to building the cluster:

- **Docker:** Engine for running containerized nodes.
- **kind:** Cluster provisioning tool.
- **kubectl:** Client for interacting with the Kubernetes API.
- **Helm:** Package manager for installing Cilium manifests.
- **Cilium CLI:** Tool for managing and validating the network configuration.

### macOS (Homebrew)
```bash
brew install kind kubectl helm cilium-cli
```

### Linux
```bash
# 1. Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# 2. Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 3. Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
```

## Procedure

### 1. Create the Configuration File

`kind` must be configured to boot without its default Container Network Interface (CNI). The following configuration specifies one control-plane node and two worker nodes to simulate a multi-node environment suitable for testing cross-node routing.

```bash
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
EOF
```

### 2. Boot the Cluster

Initialize the cluster using the generated configuration file:

```bash
kind create cluster --config kind-config.yaml
```

Once provisioning completes, verify the cluster status. The nodes will appear with a `NotReady` status because the networking component has not yet been installed.

```bash
kubectl get nodes
```

### 3. Install the eBPF Network

Deploy Cilium to provide the required networking functionality:

```bash
cilium install
```

This command deploys the Cilium DaemonSet. The Cilium agents boot on the nodes, compile the eBPF C-code, and attach it to the virtual network interfaces.

Monitor the rollout status:

```bash
cilium status --wait
```

### 4. Verify Node Status

After the Cilium installation completes successfully, re-check the node status:

```bash
kubectl get nodes
```

All nodes should now report a `Ready` status, indicating the network is fully operational.

### 5. Connectivity Test

To confirm that packets can successfully travel across node boundaries, run the automated connectivity test suite:

```bash
cilium connectivity test
```

This process takes a few minutes, deploying multiple test Pods to perform routing and connectivity checks. A successful outcome confirms a fully functional Kubernetes environment.

---

## Alternative: Cilium Installation via Helm

As an alternative to the CLI, Helm can be used to install Cilium.

1. Add and update the Cilium Helm repository:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

2. Execute the Helm installation:

```bash
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set ipam.mode=kubernetes
```
