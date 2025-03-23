# Kubernetes Storage

## Overview Kubernetes Volumes
Kubernetes **Volumes** provide persistent or ephemeral storage for pods. Unlike container storage, volumes **persist** across container restarts within a pod.

---

## 1. **Types of Volumes**
| Volume Type | Description |
|-------------|-------------|
| `emptyDir` | Temporary storage that is deleted when the pod is removed. |
| `hostPath` | Mounts a directory from the host node. |
| `configMap` | Mounts a ConfigMap as a file system. |
| `secret` | Mounts a Secret as a file system. |
| `persistentVolumeClaim (PVC)` | Uses a PersistentVolume (PV) for long-term storage. |
| `nfs` | Mounts an NFS (Network File System) share. |
| `awsElasticBlockStore (EBS)` | Mounts an AWS EBS volume. |
| `azureDisk` | Mounts an Azure managed disk. |

---

## 2. **Using `emptyDir` (Temporary Storage)**
Data is deleted when the pod is removed.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: temp-storage
  volumes:
    - name: temp-storage
      emptyDir: {}
```
✅ Use for temporary cache, logs, or ephemeral data.  
❌ Data is lost if the pod is deleted.

---

## 3. **Using `hostPath` (Host Storage)**
Mounts a folder from the **host node**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: host-storage
  volumes:
    - name: host-storage
      hostPath:
        path: /var/data
```
✅ Useful for node-local storage.  
❌ Not suitable for multi-node clusters.

---

## 4. **Using `configMap` (Mount Config as File)**
Mounts a ConfigMap as a file inside the container.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.json: |
    { "key": "value" }
---
apiVersion: v1
kind: Pod
metadata:
  name: configmap-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /etc/config
          name: config-volume
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```
✅ Configuration as files, instead of environment variables.  
✅ Config changes can be applied without restarting the pod.

---

## 5. **Using `secret` (Mount Secrets as File)**
Mounts Kubernetes Secrets securely.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  password: cGFzc3dvcmQ=  # Base64 encoded "password"
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /etc/secret
          name: secret-volume
  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
```
✅ Secure way to provide sensitive data.  
✅ Mounted secrets are automatically updated.

---

## 6. **Using `persistentVolumeClaim (PVC)`**
PVC requests storage from a PersistentVolume (PV).

### **Step 1: Create a PersistentVolume (PV)**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

### **Step 2: Create a PersistentVolumeClaim (PVC)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### **Step 3: Use PVC in a Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: persistent-storage
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: pvc-example
```
✅ Data persists even after pod restarts.  
✅ Best for databases and stateful applications.  

---

## 7. **Using `nfs` (Network File System)**
Mounts an NFS share from a remote server.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: nfs-storage
  volumes:
    - name: nfs-storage
      nfs:
        server: 192.168.1.100
        path: /exported/data
```
✅ Allows multiple pods to share the same storage.  
✅ Useful for distributed storage solutions.  

---

## 8. **Using `awsElasticBlockStore (EBS)`**
Mounts an AWS EBS volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ebs-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: ebs-storage
  volumes:
    - name: ebs-storage
      awsElasticBlockStore:
        volumeID: vol-0123456789abcdef0
        fsType: ext4
```
✅ Supports persistent storage in AWS.  
❌ Works only with **AWS** infrastructure.

---

## 9. **Using `azureDisk` (Azure Managed Disk)**
Mounts an Azure disk.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: azure-disk-example
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: azure-storage
  volumes:
    - name: azure-storage
      azureDisk:
        diskName: my-managed-disk
        diskURI: /subscriptions/xxx/resourceGroups/xxx/providers/Microsoft.Compute/disks/my-managed-disk
```
✅ Best for stateful workloads on **Azure**.

---

## 10. **Check Volume Usage**
### **List All Persistent Volumes (PV)**
```sh
kubectl get pv
```

### **List All Persistent Volume Claims (PVC)**
```sh
kubectl get pvc
```

### **Describe a Volume in a Pod**
```sh
kubectl describe pod pvc-example
```

---

## **Best Practices**
✅ **Use `emptyDir` only for temporary data.**  
✅ **Use `configMap` and `secret` for non-sensitive & sensitive configurations.**  
✅ **Use `pvc` for persistent storage to avoid data loss.**  
✅ **Ensure volumes match your cloud provider (EBS, AzureDisk, NFS).**  
✅ **Mount only necessary storage to limit security risks.**  

---

## **Summary**
| Volume Type | Use Case |
|-------------|------------------------------|
| `emptyDir` | Temporary storage (cache, logs). |
| `hostPath` | Access host system files (local storage). |
| `configMap` | Mount configuration files. |
| `secret` | Securely mount sensitive data. |
| `persistentVolumeClaim (PVC)` | Persistent storage for stateful apps. |
| `nfs` | Shared storage across multiple pods. |
| `awsElasticBlockStore` | Persistent storage in AWS. |
| `azureDisk` | Persistent storage in Azure. |

---