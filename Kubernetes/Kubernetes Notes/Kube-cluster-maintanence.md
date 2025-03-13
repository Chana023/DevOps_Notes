# Cluster Maintenance




## OS Upgrade Overview
OS upgrades in a Kubernetes cluster involve updating the operating system of worker nodes **without disrupting workloads**. The key methods are:
1. **Rolling Node Replacements** (Recommended)
2. **In-Place OS Upgrades** (Riskier)
3. **Using Kubernetes Drain & Cordon for Safe Upgrades**

---

## 1. Rolling Node Replacements (Recommended)

### Step 1: **Add New Nodes with the Upgraded OS**
- If using **cloud-based Kubernetes**, create new nodes with the upgraded OS.
- If using **on-premise Kubernetes**, manually provision a new node.

### Step 2: **Drain the Old Node**
```sh
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
- This **evicts all pods** from the node **except DaemonSets**.

### Step 3: **Remove the Old Node**
```sh
kubectl delete node <node-name>
```

### Step 4: **Verify the New Node is Ready**
```sh
kubectl get nodes
```

---

## 2. In-Place OS Upgrade (Riskier)

### Step 1: **Cordon the Node to Prevent Scheduling**
```sh
kubectl cordon <node-name>
```
- This **marks the node as unschedulable**.

### Step 2: **Drain Workloads**
```sh
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
- Moves workloads off the node.

### Step 3: **Perform the OS Upgrade**
- **Ubuntu/Debian:**
  ```sh
  sudo apt update && sudo apt upgrade -y
  sudo reboot
  ```
- **CentOS/RHEL:**
  ```sh
  sudo yum update -y
  sudo reboot
  ```

### Step 4: **Uncordon the Node**
```sh
kubectl uncordon <node-name>
```
- Allows scheduling on the node again.

---

## 3. Kubernetes Upgrade Alongside OS Upgrade (if needed)

If the OS upgrade requires a Kubernetes upgrade:

### Step 1: **Check the Current Version**
```sh
kubectl version
```

### Step 2: **Upgrade `kubelet` and `kubectl` (if needed)**
```sh
apt update && apt install -y kubelet=<new-version> kubectl=<new-version>
systemctl restart kubelet
```

### Step 3: **Verify Node Readiness**
```sh
kubectl get nodes
```

---

## 4. Automated OS Upgrades in Managed Kubernetes (Cloud)

- **GKE:** Use **Node Auto-Upgrade**.
- **EKS:** Upgrade nodes using **Managed Node Groups**.
- **AKS:** Use **Node Pool Upgrades**.

Example for AKS:
```sh
az aks nodepool upgrade --resource-group myResourceGroup --cluster-name myAKSCluster --name myNodePool
```

---

## 5. Monitoring Node Upgrades

### Check Node Status:
```sh
kubectl get nodes -o wide
```

### View Pod Scheduling:
```sh
kubectl get pods -A -o wide
```

### Check Events:
```sh
kubectl describe node <node-name>
```

---

## Important Notes
- **Rolling replacements** are safer than in-place upgrades.
- **Always drain nodes before upgrading** to prevent downtime.
- **Test OS upgrades in a staging environment first.**
- **Monitor logs and resource utilization post-upgrade.**