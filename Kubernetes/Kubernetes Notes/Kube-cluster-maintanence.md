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


## Cluster Upgrade Overview
Upgrading a Kubernetes cluster involves updating the **control plane** (API server, scheduler, etcd) and **worker nodes** to a newer Kubernetes version. The process depends on whether you're using **managed Kubernetes (e.g., AKS, EKS, GKE)** or a **self-managed cluster (e.g., kubeadm, kops, or RKE)**.

---

## 1. **Pre-Upgrade Checklist**
Before upgrading, ensure:
1. **Check Current Version:**
   ```sh
   kubectl version --short
   kubectl get nodes
   ```
2. **Check Available Upgrades:**
   ```sh
   kubeadm upgrade plan
   ```
3. **Backup etcd (for self-managed clusters):**
   ```sh
   ETCDCTL_API=3 etcdctl snapshot save backup.db --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
   ```
4. **Drain a Control Plane Node Before Upgrade:**
   ```sh
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   ```

---

## 2. **Upgrading a Self-Managed Kubernetes Cluster (kubeadm)**

### Step 1: Upgrade Control Plane Nodes

#### 1. Upgrade kubeadm:
```sh
apt update && apt install -y kubeadm=<new-version>
```

#### 2. Upgrade Kubernetes Control Plane:
```sh
kubeadm upgrade apply <new-version>
```

#### 3. Upgrade kubelet & kubectl:
```sh
apt install -y kubelet=<new-version> kubectl=<new-version>
systemctl restart kubelet
```

#### 4. Verify Control Plane Upgrade:
```sh
kubectl get nodes
kubectl version
```

---

### Step 2: Upgrade Worker Nodes

#### 1. Drain Each Worker Node:
```sh
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

#### 2. Upgrade kubeadm, kubelet, and kubectl:
```sh
apt update && apt install -y kubeadm=<new-version>
kubeadm upgrade node
apt install -y kubelet=<new-version> kubectl=<new-version>
systemctl restart kubelet
```

#### 3. Uncordon the Node:
```sh
kubectl uncordon <node-name>
```

#### 4. Verify Upgrade:
```sh
kubectl get nodes
kubectl get pods -A
```

---

## 3. **Upgrading a Managed Kubernetes Cluster**

### **Amazon EKS**
```sh
aws eks update-cluster-version --name my-cluster --kubernetes-version <new-version>
```
Upgrade worker nodes:
```sh
aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name my-nodegroup
```

### **Google Kubernetes Engine (GKE)**
```sh
gcloud container clusters upgrade my-cluster --master
gcloud container clusters upgrade my-cluster --node-pool my-nodepool
```

### **Azure Kubernetes Service (AKS)**
```sh
az aks upgrade --resource-group myResourceGroup --name myAKSCluster --kubernetes-version <new-version>
```

---

## 4. **Post-Upgrade Steps**
### 1. Verify Cluster Health:
```sh
kubectl get nodes
kubectl get pods -A
kubectl get cs
```

### 2. Check Logs for Errors:
```sh
kubectl logs -l component=kube-apiserver -n kube-system
```

### 3. Ensure DNS and Networking Work:
```sh
kubectl run test-pod --image=busybox --rm -it -- nslookup kubernetes.default
```

---

## Important Notes
- **Upgrade control plane first, then worker nodes.**
- **Drain nodes before upgrading to avoid pod disruptions.**
- **Always backup etcd before upgrading a self-managed cluster.**
- **Test upgrades in a non-production environment first.**