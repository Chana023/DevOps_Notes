Networking

## Overview
Kubernetes **Networking** enables communication between:
1. **Pods within the same node**
2. **Pods across different nodes**
3. **Pods and external services**
4. **Pods and the internet**

Unlike traditional networking, **Kubernetes networking is flat**, meaning **all pods can communicate with each other by default**.

---

## 1. **Key Networking Components**
| Component | Description |
|-----------|------------|
| **Pod-to-Pod Networking** | All pods can communicate with each other across nodes without NAT. |
| **ClusterIP (Service)** | Exposes a service inside the cluster. |
| **NodePort (Service)** | Exposes a service on each node's IP, at a static port. |
| **LoadBalancer (Service)** | Exposes a service externally using a cloud provider’s load balancer. |
| **Ingress** | Manages external HTTP/S access to services. |
| **Network Policies** | Controls traffic between pods. |
| **DNS** | Provides internal service discovery. |

---

## 2. **Pod-to-Pod Communication**
By default, Kubernetes assigns **each pod an IP address**. All pods can communicate freely, even across nodes.

### **Check Pod IPs**
```sh
kubectl get pods -o wide
```

✅ No need for **NAT or port forwarding**.  
✅ Uses **CNI (Container Network Interface)** plugins for networking (Calico, Flannel, Cilium).  

---

## 3. **Services: Exposing Applications**

### **(a) ClusterIP (Default)**
- Exposes services **internally** within the cluster.
- No external access.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-clusterip-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

🔹 **Access inside cluster:**  
```sh
curl http://my-clusterip-service:80
```

---

### **(b) NodePort**
- Exposes a service on **each node's IP**, using a fixed port (`30000-32767`).
- Can be accessed externally.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30007
  type: NodePort
```

🔹 **Access externally:**  
```sh
curl http://<NODE_IP>:30007
```

---

### **(c) LoadBalancer**
- Used in **cloud environments** (AWS, GCP, Azure).
- Exposes a service via a **cloud provider’s load balancer**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

🔹 **Get external IP:**  
```sh
kubectl get svc my-loadbalancer-service
```

✅ Used for **production environments**.  
❌ Not available in **bare-metal clusters** without MetalLB.  

---

## 4. **Ingress: HTTP/S Routing**
- Routes external HTTP/S traffic to services inside the cluster.
- Provides **virtual hosting** (multiple services on one IP).
- Requires an **Ingress Controller** (NGINX, Traefik).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-clusterip-service
                port:
                  number: 80
```

🔹 **Access externally:**  
```sh
curl http://myapp.example.com
```

✅ **SSL/TLS termination** supported via cert-manager.

---

## 5. **Network Policies: Controlling Traffic**
By default, **all pods can talk to each other**. **Network Policies** restrict access.

Example: **Allow only frontend to talk to backend**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

✅ **Enhances security** by limiting communication.

---

## 6. **DNS: Service Discovery**
Kubernetes has an internal DNS service that resolves **service names to IPs**.

Example: A pod can access `my-service` directly:
```sh
curl http://my-service:80
```

✅ DNS is **enabled by default** via `CoreDNS`.

---

## 7. **CNI (Container Network Interface) Plugins**
Kubernetes uses **CNI plugins** to manage pod networking.

| CNI Plugin | Features |
|------------|----------|
| **Calico** | Network policies, BGP routing. |
| **Flannel** | Simple overlay network. |
| **Cilium** | Security & observability using eBPF. |
| **WeaveNet** | Encrypted networking. |
| **Kube-router** | BGP-based networking. |

🔹 **Check CNI in your cluster:**
```sh
kubectl get pods -n kube-system
```

---

## 8. **Port Forwarding to a Pod**
To access a pod locally:
```sh
kubectl port-forward pod/mypod 8080:80
```
✅ Useful for debugging.

---

## 9. **Checking Network Connectivity**
### **Check all services**
```sh
kubectl get svc
```

### **Check pod IPs**
```sh
kubectl get pods -o wide
```

### **Test connectivity between pods**
```sh
kubectl exec -it mypod -- curl http://other-pod-ip:80
```

### **Check Ingress controller logs**
```sh
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

---

## **Best Practices**
✅ **Use Ingress instead of NodePort for HTTP traffic.**  
✅ **Apply Network Policies to restrict traffic.**  
✅ **Use a CNI plugin that supports your needs (Calico, Flannel).**  
✅ **Monitor networking issues with `kubectl logs` and `kubectl describe`.**  
✅ **Avoid exposing pods directly—use Services and Ingress.**  

---

## **Summary**
| Feature | Description |
|---------|------------|
| **Pod-to-Pod** | All pods can communicate by default. |
| **ClusterIP (Service)** | Exposes service only inside the cluster. |
| **NodePort (Service)** | Exposes service on each node’s IP. |
| **LoadBalancer (Service)** | Exposes service via cloud load balancer. |
| **Ingress** | HTTP/S routing with virtual hosting. |
| **Network Policies** | Restrict traffic between pods. |
| **DNS** | Automatic service discovery. |
| **CNI Plugins** | Manages pod networking (Calico, Flannel, Cilium). |

---


# CNI (Container Network Interface) Plugins

## Overview
Container Network Interface (**CNI**) plugins enable **pod-to-pod** communication in Kubernetes.  
They implement networking features like **IP allocation, routing, and network policies**.

Kubernetes **requires a CNI plugin** to function properly.

---

## 1. **How CNI Works in Kubernetes**
- Each **pod** gets its own **IP address**.
- CNI plugins **assign and manage IP addresses** for pods.
- Handles **routing traffic** between pods across nodes.

🔹 **Check CNI plugin in your cluster:**
```sh
kubectl get pods -n kube-system
```

---

## 2. **Popular CNI Plugins**
| CNI Plugin  | Features |
|-------------|----------|
| **Calico**  | Network policies, BGP routing, security. |
| **Flannel** | Simple overlay network using VXLAN. |
| **Cilium**  | eBPF-based security & observability. |
| **WeaveNet** | Encrypted networking, automatic topology detection. |
| **Kube-Router** | BGP-based networking, minimal overhead. |
| **Multus**  | Supports multiple CNI plugins per pod. |
| **Canal**   | Combines Flannel & Calico (for policies + VXLAN). |

---

## 3. **Calico (Best for Network Policies)**
- Uses **BGP (Border Gateway Protocol)** for routing.
- Supports **Network Policies** (restrict pod traffic).
- Works in **bare-metal, cloud, and hybrid environments**.

🔹 **Install Calico**:
```sh
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

🔹 **Example Network Policy (Deny All Traffic)**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

✅ **Best for security-focused environments.**  
✅ **Works with BGP and VXLAN.**  
❌ **More complex than Flannel.**  

---

## 4. **Flannel (Simplest Overlay Network)**
- Uses **VXLAN** to create an overlay network.
- **No network policies** (use with Calico for policies).
- Best for **simple, internal networking**.

🔹 **Install Flannel**:
```sh
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

🔹 **Check Flannel pods**:
```sh
kubectl get pods -n kube-system | grep flannel
```

✅ **Easy to set up** and lightweight.  
✅ **Works in cloud and on-prem.**  
❌ **No built-in security policies.**  

---

## 5. **Cilium (Best for Security & Observability)**
- Uses **eBPF** instead of iptables for networking.
- **High-performance** (low CPU/memory usage).
- Built-in **network policies** and **observability tools**.

🔹 **Install Cilium**:
```sh
helm install cilium cilium/cilium --namespace kube-system
```

🔹 **Enable Hubble (for network observability)**:
```sh
cilium hubble enable
```

✅ **Best for high-scale clusters**.  
✅ **Observability and security built-in**.  
❌ **More complex setup**.  

---

## 6. **Weave Net (Best for Simplicity & Encryption)**
- **Automatic network topology detection**.
- **Encrypted pod-to-pod communication**.
- Supports **Network Policies**.

🔹 **Install WeaveNet**:
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

🔹 **Check WeaveNet pods**:
```sh
kubectl get pods -n kube-system | grep weave
```

✅ **Simple setup with encryption**.  
✅ **Works across clouds**.  
❌ **Higher CPU/memory usage**.  

---

## 7. **Kube-Router (Best for Performance)**
- Uses **BGP routing** (like Calico).
- Minimal overhead (no overlay networking).
- Supports **Network Policies**.

🔹 **Install Kube-Router**:
```sh
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kube-router.yaml
```

🔹 **Check Kube-Router pods**:
```sh
kubectl get pods -n kube-system | grep kube-router
```

✅ **Best for performance-sensitive applications**.  
✅ **Built-in network policies**.  
❌ **Requires BGP knowledge**.  

---

## 8. **Multus (Best for Multi-CNI Support)**
- Allows using **multiple CNI plugins** per pod.
- Needed for **advanced networking (e.g., SR-IOV, DPDK).**
- **Not a standalone CNI**—used **with** other plugins.

🔹 **Install Multus**:
```sh
kubectl apply -f https://github.com/k8snetworkplumbingwg/multus-cni/releases/latest/download/multus-daemonset.yml
```

✅ **Best for complex networking (5G, AI workloads).**  
❌ **Not needed for basic clusters.**  

---

## 9. **Canal (Combines Flannel & Calico)**
- Uses **Flannel** for networking.
- Uses **Calico** for **Network Policies**.
- Best **hybrid option**.

🔹 **Install Canal**:
```sh
kubectl apply -f https://docs.projectcalico.org/manifests/canal.yaml
```

✅ **Best mix of performance & security**.  
✅ **Works in hybrid environments**.  

---

## 10. **Comparing CNI Plugins**
| Plugin       | Network Policies | Encryption | Performance | Best For |
|-------------|-----------------|------------|-------------|----------|
| **Calico**  | ✅ Yes | ❌ No | ⚡ High | Security-focused clusters |
| **Flannel** | ❌ No | ❌ No | ⚡ High | Simple setups |
| **Cilium**  | ✅ Yes | ✅ Yes | 🚀 Very High | High-performance networking |
| **WeaveNet** | ✅ Yes | ✅ Yes | 🔥 Medium | Encrypted pod communication |
| **Kube-Router** | ✅ Yes | ❌ No | ⚡ High | Low-latency workloads |
| **Multus** | N/A | ❌ No | 🚀 Very High | Multi-CNI networking |
| **Canal** | ✅ Yes | ❌ No | ⚡ High | Hybrid security + performance |

---

## 11. **Choosing the Right CNI**
| Use Case | Recommended CNI |
|----------|----------------|
| Simple setup (internal traffic) | **Flannel** |
| Strong security & policies | **Calico** |
| High-performance networking | **Cilium** |
| Encrypted communication | **WeaveNet** |
| BGP routing & minimal overhead | **Kube-Router** |
| Advanced multi-CNI setup | **Multus** |
| Balanced security & performance | **Canal** |

---

## **Checking CNI in Kubernetes**
### **List all pods in `kube-system` namespace:**
```sh
kubectl get pods -n kube-system
```

### **Check which CNI is installed:**
```sh
kubectl get daemonsets -n kube-system
```

### **Check CNI logs (example for Calico):**
```sh
kubectl logs -n kube-system -l k8s-app=calico-node
```

---

## **Best Practices**
✅ **Use a CNI plugin that fits your security & performance needs.**  
✅ **Monitor CNI performance using metrics tools (Prometheus, Hubble for Cilium).**  
✅ **Use Network Policies to secure pod traffic.**  
✅ **Test pod-to-pod communication (`kubectl exec` + `curl`).**  
✅ **Keep CNI plugins updated for security patches.**  

---

## **Summary**
- **CNI plugins** manage Kubernetes networking.
- **Flannel** (simple), **Calico** (security), **Cilium** (performance).
- **Use Network Policies** for security.
- **Monitor networking** using logs & metrics.

---