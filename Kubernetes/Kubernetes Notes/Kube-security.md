# Kubernetes Security

## Overview TLS Certificates in Kube
Kubernetes uses **TLS certificates** for:
1. **Securing communication** between components (API server, kubelet, etcd).
2. **Securing ingress traffic** (TLS termination for applications).
3. **Encrypting secrets** using a **custom TLS certificate**.

---

## 1. **Kubernetes Control Plane TLS Certificates**

Kubernetes manages control plane TLS certificates in `/etc/kubernetes/pki/`.

### **View Expiring Certificates**
```sh
kubectl get csr
sudo kubeadm certs check-expiration
```

### **Renew Expired Certificates**
```sh
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

### **Manually Create a New TLS Certificate**
```sh
openssl req -new -newkey rsa:2048 -nodes -keyout server.key -out server.csr -subj "/CN=my-cluster"
openssl x509 -req -in server.csr -signkey server.key -out server.crt -days 365
```

---

## 2. **TLS in Ingress (Securing Applications)**

### **Create a Self-Signed TLS Certificate**
```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=my-app"
```

### **Create a Kubernetes Secret for TLS**
```sh
kubectl create secret tls my-tls-secret --cert=tls.crt --key=tls.key
```

### **Use TLS in an Ingress Resource**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - my-app.example.com
      secretName: my-tls-secret
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

### **Apply the Ingress Configuration**
```sh
kubectl apply -f ingress.yaml
```

---

## 3. **TLS in Kubernetes Secrets (For Applications Using HTTPS)**

### **Create a TLS Secret**
```sh
kubectl create secret tls my-app-tls --cert=tls.crt --key=tls.key
```

### **Reference the TLS Secret in a Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: my-app
      image: nginx
      volumeMounts:
        - name: tls-certs
          mountPath: "/etc/tls"
          readOnly: true
  volumes:
    - name: tls-certs
      secret:
        secretName: my-app-tls
```

---

## 4. **Automated TLS Management with Cert-Manager**
[cert-manager](https://cert-manager.io) automates the issuance of TLS certificates using **Let's Encrypt** or custom CAs.

### **Install cert-manager**
```sh
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
```

### **Issue a TLS Certificate with Let's Encrypt**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-app-cert
spec:
  secretName: my-app-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - my-app.example.com
```

### **Apply and Verify**
```sh
kubectl apply -f certificate.yaml
kubectl get certificates
kubectl describe certificate my-app-cert
```

---

## 5. **Checking and Renewing Certificates**

### **List TLS Secrets**
```sh
kubectl get secrets
```

### **Check Certificate Expiry**
```sh
openssl x509 -in tls.crt -noout -enddate
```

### **Renew Cert-Manager Certificates**
```sh
kubectl delete certificate my-app-cert
kubectl apply -f certificate.yaml
```

---

## Important Notes
- **Kubernetes control plane certificates expire after 1 year; monitor them!**
- **Use Let's Encrypt and cert-manager for automatic TLS management.**
- **Always use TLS secrets instead of storing certs in ConfigMaps.**

## Certificates API Overview
The **Kubernetes Certificates API** is used for:
- Requesting and issuing **TLS certificates** for workloads.
- Managing **certificate signing requests (CSRs)** for node authentication.
- Automating the **approval and signing** of certificates.

---

## 1. **Creating a Certificate Signing Request (CSR)**

### **Step 1: Generate a Private Key and CSR**
```sh
openssl genrsa -out my-key.key 2048
openssl req -new -key my-key.key -subj "/CN=my-app" -out my-csr.csr
```

### **Step 2: Encode CSR in Base64**
```sh
cat my-csr.csr | base64 | tr -d '\n'
```

### **Step 3: Create a Kubernetes CSR Resource**
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: my-app-csr
spec:
  request: <BASE64_ENCODED_CSR>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - digital signature
    - key encipherment
    - client auth
```

### **Apply the CSR**
```sh
kubectl apply -f csr.yaml
kubectl get csr
```

---

## 2. **Approving and Signing a CSR**

### **Manually Approve a CSR**
```sh
kubectl certificate approve my-app-csr
```

### **Deny a CSR**
```sh
kubectl certificate deny my-app-csr
```

---

## 3. **Retrieve and Use the Signed Certificate**

### **Check CSR Status**
```sh
kubectl get csr my-app-csr -o yaml
```

### **Download Signed Certificate**
```sh
kubectl get csr my-app-csr -o jsonpath='{.status.certificate}' | base64 -d > my-app.crt
```

### **Use the Signed Certificate in a Secret**
```sh
kubectl create secret tls my-app-tls --cert=my-app.crt --key=my-key.key
```

---

## 4. **Automatically Signing CSRs with a Controller**

For automating approval:
1. Create a **Kubernetes controller** to watch for CSRs.
2. Use an admission controller to validate requests.
3. Auto-approve CSRs based on custom rules.

Example:
```sh
kubectl get csr -o json | jq '.items[] | {name: .metadata.name, approved: .status.conditions}'
```

---

## 5. **Checking and Managing Certificates**

### **List All CSRs**
```sh
kubectl get csr
```

### **Describe a Specific CSR**
```sh
kubectl describe csr my-app-csr
```

### **Delete a CSR**
```sh
kubectl delete csr my-app-csr
```

---

## Important Notes
- **CSRs must be manually approved unless an automated controller is in place.**
- **Use the Kubernetes API Server as a signer or integrate with an external CA.**
- **Signed certificates should be stored in Kubernetes secrets for security.**

## Overview Kubernetes Kubeconfig
The **kubeconfig** file is used by `kubectl` to authenticate and interact with a Kubernetes cluster. It contains details about:
- **Clusters** (API server details)
- **Users** (authentication credentials)
- **Contexts** (combination of a cluster, a user, and a namespace)

The default kubeconfig file is located at:
```sh
~/.kube/config
```

---

## 1. **View Current Kubeconfig**
```sh
kubectl config view
```

---

## 2. **Setting Up a Kubeconfig File**

### **Basic Kubeconfig Structure**
```yaml
apiVersion: v1
kind: Config
clusters:
  - name: my-cluster
    cluster:
      server: https://my-cluster-api-server:6443
      certificate-authority-data: <BASE64_CA_CERT>
users:
  - name: my-user
    user:
      client-certificate-data: <BASE64_CERT>
      client-key-data: <BASE64_KEY>
contexts:
  - name: my-context
    context:
      cluster: my-cluster
      user: my-user
      namespace: default
current-context: my-context
```

---

## 3. **Setting and Switching Contexts**

### **View Available Contexts**
```sh
kubectl config get-contexts
```

### **Set the Current Context**
```sh
kubectl config use-context my-context
```

### **Create a New Context**
```sh
kubectl config set-context my-new-context --cluster=my-cluster --user=my-user --namespace=my-namespace
```

---

## 4. **Managing Multiple Kubeconfig Files**

### **Set a Custom Kubeconfig File**
```sh
export KUBECONFIG=/path/to/my-kubeconfig
```

### **Merge Multiple Kubeconfig Files**
```sh
export KUBECONFIG=~/.kube/config:/path/to/another/kubeconfig
kubectl config view --merge --flatten > ~/.kube/config-merged
```

---

## 5. **Generating a Kubeconfig for a Service Account**
1. **Create a Service Account**
   ```sh
   kubectl create serviceaccount my-sa
   ```

2. **Create a Role and Bind It**
   ```sh
   kubectl create rolebinding my-sa-binding --clusterrole=view --serviceaccount=default:my-sa
   ```

3. **Get the Token for Authentication**
   ```sh
   kubectl get secret $(kubectl get sa my-sa -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
   ```

4. **Generate a Kubeconfig for the Service Account**
   ```yaml
   apiVersion: v1
   kind: Config
   clusters:
     - name: my-cluster
       cluster:
         server: https://my-cluster-api-server:6443
         certificate-authority-data: <BASE64_CA_CERT>
   users:
     - name: my-sa
       user:
         token: <TOKEN>
   contexts:
     - name: my-sa-context
       context:
         cluster: my-cluster
         user: my-sa
         namespace: default
   current-context: my-sa-context
   ```

---

## 6. **Testing Kubeconfig Authentication**
```sh
kubectl --kubeconfig=/path/to/kubeconfig get nodes
```

---

## 7. **Deleting a Context or User**
### **Remove a Context**
```sh
kubectl config delete-context my-context
```

### **Remove a Cluster**
```sh
kubectl config delete-cluster my-cluster
```

### **Remove a User**
```sh
kubectl config delete-user my-user
```

---

## Important Notes
- **Avoid modifying the kubeconfig manually; use `kubectl config set-*` commands.**
- **Always use `export KUBECONFIG=<file>` for temporary kubeconfig changes.**
- **Service accounts should be used for automation and restricted access.**