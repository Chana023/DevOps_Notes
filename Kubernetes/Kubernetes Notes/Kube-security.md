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

## Overview Kubernetes API Groups
Kubernetes API groups organize related API objects, making it easier to extend functionality without affecting core components. The **API server** exposes resources under different groups.

### **Types of API Groups**
1. **Core API Group (`apiVersion: v1`)**
2. **Named API Groups (e.g., `apps`, `batch`, `rbac.authorization.k8s.io`)**
3. **Custom Resource Definitions (CRDs) (`apiextensions.k8s.io`)**

---

## 1. **Core API Group (`apiVersion: v1`)**
The **Core API** has no group prefix and contains essential Kubernetes objects.

### **Example Core API Resources**
```sh
kubectl api-resources --api-group=""
```
**Common Core API Objects:**
- Pods (`kind: Pod`)
- Services (`kind: Service`)
- ConfigMaps (`kind: ConfigMap`)
- Secrets (`kind: Secret`)
- Namespaces (`kind: Namespace`)

### **Example YAML**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

---

## 2. **Named API Groups**
Named API groups organize Kubernetes resources for better structure.

### **Listing API Groups**
```sh
kubectl api-versions
```

### **Common API Groups**
| API Group                        | Example Resource         | Usage |
|-----------------------------------|-------------------------|----------------|
| `apps`                            | `Deployment`            | Manages workloads |
| `batch`                           | `Job`, `CronJob`        | Handles batch jobs |
| `autoscaling`                     | `HorizontalPodAutoscaler` | Manages scaling |
| `rbac.authorization.k8s.io`       | `Role`, `RoleBinding`   | Role-based access control |
| `networking.k8s.io`               | `Ingress`, `NetworkPolicy` | Network management |
| `apiextensions.k8s.io`            | `CustomResourceDefinition` | Extends Kubernetes API |

---

## 3. **Example YAML for Named API Groups**

### **Deployment (API Group: `apps/v1`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-container
          image: nginx
```

### **Ingress (API Group: `networking.k8s.io/v1`)**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: my-app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

---

## 4. **Custom Resource Definitions (CRDs)**
CRDs extend Kubernetes with **custom API resources**.

### **List Installed CRDs**
```sh
kubectl get crds
```

### **Example CRD Definition (`apiextensions.k8s.io/v1`)**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: myresources
    singular: myresource
    kind: MyResource
```

---

## 5. **Interacting with API Groups**
### **Get Available API Resources**
```sh
kubectl api-resources
```

### **Check API Versions for a Specific Resource**
```sh
kubectl explain deployment --api-version=apps/v1
```

### **Accessing API Directly with `kubectl get`**
```sh
kubectl get deployments.v1.apps
kubectl get jobs.batch
kubectl get networkpolicies.networking.k8s.io
```

---

## 6. **Checking and Debugging API Calls**
### **Get Raw API Response**
```sh
kubectl get --raw /apis/apps/v1
```

### **Describe API Group Details**
```sh
kubectl explain pod --api-version=v1
```

---

## Important Notes
- **Core API objects do not have a group (`apiVersion: v1`).**
- **API groups are versioned (`v1`, `v1beta1`) to support upgrades.**
- **Use `kubectl api-resources` to list all available objects and groups.**

## Overview Kubernetes Authorization
Kubernetes **authorization** controls what actions users, service accounts, and processes can perform within the cluster. It ensures **RBAC (Role-Based Access Control)**, **ABAC (Attribute-Based Access Control)**, and other mechanisms are enforced.

### **Types of Authorization in Kubernetes**
1. **RBAC (Role-Based Access Control)** – Assigns roles to users, groups, or service accounts.
2. **ABAC (Attribute-Based Access Control)** – Uses policies for granular access (less common).
3. **Webhook Authorization** – External authorization through API calls.
4. **Node Authorization** – Restricts node permissions to necessary actions.

---

## 1. **Role-Based Access Control (RBAC)**

### **View Current Authorization Mode**
```sh
kubectl api-versions | grep authorization
```

### **List Roles and RoleBindings**
```sh
kubectl get roles,rolebindings --all-namespaces
kubectl get clusterroles,clusterrolebindings
```

### **Example: Creating a Role and RoleBinding**

#### **Define a Role (Namespace-Specific)**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

#### **Bind the Role to a User**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### **Apply the RBAC Configurations**
```sh
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
```

---

## 2. **Cluster-Wide Authorization (ClusterRole & ClusterRoleBinding)**

### **Example: Granting Read-Only Access to All Namespaces**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

#### **Bind ClusterRole to a User**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-pod-reader-binding
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### **Apply the ClusterRole Configurations**
```sh
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml
```

---

## 3. **Verifying User Permissions**

### **Check If a User Has Permissions**
```sh
kubectl auth can-i get pods --as alice
kubectl auth can-i delete deployments --as bob --namespace=dev
```

### **Get Detailed User Access**
```sh
kubectl auth can-i --list
```

---

## 4. **Service Account Authorization**

### **Create a Service Account**
```sh
kubectl create serviceaccount my-service-account
```

### **Bind Service Account to a Role**
```sh
kubectl create rolebinding sa-pod-reader-binding --role=pod-reader --serviceaccount=default:my-service-account
```

### **Use a Service Account in a Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-sa
spec:
  serviceAccountName: my-service-account
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
```

---

## 5. **Webhook Authorization**
- **Custom external authorization**
- Kubernetes calls an external API to determine access.

### **Enable Webhook Authorization in API Server**
```sh
kube-apiserver --authorization-mode=Webhook
```

### **Example Webhook Policy (External API)**
```json
{
  "apiVersion": "authorization.k8s.io/v1",
  "kind": "SubjectAccessReview",
  "spec": {
    "user": "alice",
    "resourceAttributes": {
      "namespace": "default",
      "verb": "get",
      "resource": "pods"
    }
  }
}
```

---

## 6. **Node Authorization**
- **Used for kubelet permissions** (e.g., scheduling workloads on nodes).
- Allows nodes to manage their assigned pods.

### **Enable Node Authorization in API Server**
```sh
kube-apiserver --authorization-mode=Node
```

---

## 7. **Disabling Authorization (Not Recommended)**
- Only use for testing purposes.

```sh
kube-apiserver --authorization-mode=AlwaysAllow
```

---

## Important Notes
- **RBAC is the most common authorization method in Kubernetes.**
- **Use `kubectl auth can-i` to troubleshoot authorization issues.**
- **Service accounts should have the minimum required permissions.**
- **Webhook authorization is useful for external policy enforcement.**

## Overview Kubernetes Service Accounts
A **Service Account** in Kubernetes is used by **pods** to authenticate against the Kubernetes API. By default, each pod uses the `default` service account in its namespace unless a different one is specified.

### **Key Features**
- Used by **pods** to interact with the Kubernetes API.
- Associated with **secrets** for authentication.
- Can be linked to **RBAC roles** for fine-grained access control.

---

## 1. **Listing Service Accounts**

### **List All Service Accounts in a Namespace**
```sh
kubectl get serviceaccounts -n default
```

### **View Details of a Specific Service Account**
```sh
kubectl describe serviceaccount my-sa -n default
```

---

## 2. **Creating a Service Account**

### **Using `kubectl create`**
```sh
kubectl create serviceaccount my-sa
```

### **Using a YAML Manifest**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
  namespace: default
```

```sh
kubectl apply -f my-sa.yaml
```

---

## 3. **Assigning a Service Account to a Pod**

By default, a pod uses the `default` service account. To specify a different service account:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-sa
  containers:
    - name: my-container
      image: nginx
```

```sh
kubectl apply -f my-pod.yaml
```

---

## 4. **Binding a Service Account to RBAC Roles**

### **Create a Role**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

### **Bind the Role to the Service Account**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```sh
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
```

---

## 5. **Retrieving the Token of a Service Account**
Service accounts are associated with secrets that contain authentication tokens.

### **Get the Secret Name**
```sh
kubectl get secret -n default | grep my-sa
```

### **Retrieve the Token**
```sh
kubectl get secret <secret-name> -o jsonpath="{.data.token}" | base64 --decode
```

---

## 6. **Using a Service Account Outside the Cluster**
To authenticate an external application using a service account:

1. **Extract the token** using the above method.
2. **Set the token in the `kubeconfig` file**:

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

## 7. **Deleting a Service Account**
```sh
kubectl delete serviceaccount my-sa
```

---

## Important Notes
- **Pods inherit permissions from the assigned service account.**
- **Avoid using the default service account for security reasons.**
- **Always bind service accounts to RBAC roles to limit access.**
- **Use service accounts for applications that need to interact with the Kubernetes API.**

## Overview Kubernetes Image Security
Kubernetes **image security** ensures that container images used in pods are **trusted**, **verified**, and **free from vulnerabilities**. It involves techniques like **image signing**, **scanning**, **restricting registries**, and **pull policies**.

---

## 1. **Enforcing Image Pull Policies**
Kubernetes provides pull policies to control how images are fetched.

### **Available Pull Policies**
| Policy                | Description |
|-----------------------|-------------|
| `Always`             | Always pulls the latest image from the registry. |
| `IfNotPresent`       | Pulls the image only if it is not present on the node. |
| `Never`             | Does not pull the image; must be present on the node. |

### **Example Pod Using `Always` Pull Policy**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
    - name: my-container
      image: myregistry.com/myimage:latest
      imagePullPolicy: Always
```

---

## 2. **Using Private Image Registries**
To pull images from a **private registry**, Kubernetes needs authentication.

### **Creating a Secret for a Private Registry**
```sh
kubectl create secret docker-registry my-registry-secret \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

### **Using the Secret in a Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-pod
spec:
  imagePullSecrets:
    - name: my-registry-secret
  containers:
    - name: my-container
      image: myregistry.com/private-image:latest
```

---

## 3. **Scanning Images for Vulnerabilities**
Before deploying images, **scan for vulnerabilities** using security tools:

### **Popular Image Scanners**
- [Trivy](https://github.com/aquasecurity/trivy)
- [Clair](https://github.com/quay/clair)
- [Anchore](https://github.com/anchore/grype)

### **Scanning an Image with Trivy**
```sh
trivy image myregistry.com/myimage:latest
```

---

## 4. **Restricting Untrusted Image Registries**
To prevent pulling images from untrusted sources, configure **Admission Controllers**.

### **Example: Restrict to Trusted Registries**
Use **OPA Gatekeeper** or **Kyverno** to enforce policies.

#### **Kyverno Policy to Restrict Registries**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-image-registry
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Only images from myregistry.com are allowed."
        pattern:
          spec:
            containers:
              - image: "myregistry.com/*"
```

---

## 5. **Enabling Image Signing and Verification**
To **prevent tampered images**, use signing mechanisms like **Cosign**.

### **Signing an Image with Cosign**
```sh
cosign sign --key cosign.key myregistry.com/myimage:latest
```

### **Verifying an Image Signature**
```sh
cosign verify --key cosign.pub myregistry.com/myimage:latest
```

---

## 6. **Enforcing Non-Root Users in Containers**
Running containers as **root** increases security risks.

### **Example Pod with Non-Root User**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsNonRoot: true
  containers:
    - name: secure-container
      image: myregistry.com/myimage:latest
      securityContext:
        allowPrivilegeEscalation: false
```

---

## 7. **Preventing Privileged Containers**
A privileged container can access the host system. Avoid them for security.

### **Deny Privileged Containers Using Admission Control**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-privileged-containers
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-privileged-containers
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "false"
```

---

## 8. **Enforcing Image Policies Using Admission Controllers**
Use **Admission Controllers** to validate image sources and configurations.

### **Enable PodSecurity Admission**
```sh
kubectl label namespace default pod-security.kubernetes.io/enforce=restricted
```

---

## 9. **Best Practices**
✅ Use **trusted image registries** (Docker Hub, AWS ECR, GCR, etc.).  
✅ Enable **image scanning** before deploying images.  
✅ Use **`IfNotPresent` or `Never`** pull policies when possible.  
✅ Restrict image registries using **admission controllers**.  
✅ Enforce **image signing** for integrity verification.  
✅ Avoid **root users** and **privileged containers**.  
✅ Use **network policies** to restrict image download locations.  

---

## Summary
| Security Measure            | Purpose |
|-----------------------------|----------------------------------|
| **Pull Policies**           | Controls when images are fetched |
| **Private Registry Secrets** | Authenticates against private registries |
| **Image Scanning**          | Detects vulnerabilities before deployment |
| **Restrict Registries**     | Prevents pulling from untrusted sources |
| **Image Signing**           | Ensures image integrity |
| **Run as Non-Root**         | Prevents privilege escalation |
| **Deny Privileged Mode**    | Blocks containers with host access |
| **Admission Controllers**   | Enforces security policies |

---

## Overview Kubernetes Security Contexts
A **Security Context** in Kubernetes defines **privileges and access controls** for pods or containers. It ensures that workloads run with **least privilege**, reducing security risks.

---

## 1. **Security Context in Pods and Containers**
Security contexts can be applied at:
- **Pod Level** → Affects all containers in the pod.
- **Container Level** → Overrides the pod-level settings.

### **Example: Pod-Level Security Context**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: secure-container
      image: nginx
```

---

## 2. **Key Security Context Fields**
| Field                 | Description |
|-----------------------|-------------|
| `runAsUser`          | Runs the container as a specific **user ID (UID)**. |
| `runAsGroup`         | Runs the container as a specific **group ID (GID)**. |
| `runAsNonRoot`       | Ensures the container does not run as the root user. |
| `fsGroup`            | Sets the **file system group** for mounted volumes. |
| `allowPrivilegeEscalation` | Prevents privilege escalation (e.g., `sudo`). |
| `privileged`         | Grants full access to the host system (avoid!). |
| `readOnlyRootFilesystem` | Ensures the container file system is **read-only**. |
| `capabilities`       | Grants or removes **Linux capabilities** for containers. |
| `seccompProfile`     | Restricts **syscalls** to minimize attack surface. |

---

## 3. **Restricting Root Access**
### **Example: Preventing Root Execution**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-root-pod
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: secure-container
      image: nginx
      securityContext:
        runAsUser: 1000
```

---

## 4. **Preventing Privilege Escalation**
Blocks `setuid` or `setgid` binaries from elevating privileges.

### **Example: Denying Privilege Escalation**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  containers:
    - name: secure-container
      image: nginx
      securityContext:
        allowPrivilegeEscalation: false
```

---

## 5. **Using Read-Only Filesystem**
Prevents malware from modifying the container filesystem.

### **Example: Enforcing Read-Only Filesystem**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-fs-pod
spec:
  containers:
    - name: secure-container
      image: nginx
      securityContext:
        readOnlyRootFilesystem: true
```

---

## 6. **Dropping Linux Capabilities**
Reduces the container’s privileges by removing unnecessary Linux capabilities.

### **Example: Drop Unnecessary Capabilities**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: drop-caps-pod
spec:
  containers:
    - name: secure-container
      image: nginx
      securityContext:
        capabilities:
          drop:
            - ALL
```

---

## 7. **Restricting Privileged Mode**
Avoid using **privileged** containers as they have full access to the host.

### **Example: Denying Privileged Containers**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-privileged-pod
spec:
  containers:
    - name: secure-container
      image: nginx
      securityContext:
        privileged: false
```

---

## 8. **Setting File System Permissions**
Ensures that volumes are accessible only to specific users/groups.

### **Example: Setting `fsGroup` for Mounted Volumes**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fsgroup-pod
spec:
  securityContext:
    fsGroup: 2000
  containers:
    - name: secure-container
      image: nginx
```

---

## 9. **Enforcing Seccomp Profiles**
Limits system calls available to the container.

### **Example: Using a Seccomp Profile**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  containers:
    - name: secure-container
      image: nginx
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

---

## 10. **Verifying Security Contexts**
### **Check Security Contexts of a Pod**
```sh
kubectl get pod secure-pod -o yaml
```

### **Describe a Pod for Security Settings**
```sh
kubectl describe pod secure-pod
```

---

## **Best Practices**
✅ **Always run containers as a non-root user.**  
✅ **Use `allowPrivilegeEscalation: false` to block privilege escalation.**  
✅ **Enable `readOnlyRootFilesystem` to prevent file modifications.**  
✅ **Drop unnecessary Linux capabilities using `capabilities.drop`.**  
✅ **Restrict privileged mode (`privileged: false`).**  
✅ **Use seccomp profiles to limit available syscalls.**  

---

## Summary
| Security Measure           | Purpose |
|----------------------------|----------------------------------|
| **`runAsNonRoot`**         | Ensures the container runs as a non-root user. |
| **`allowPrivilegeEscalation: false`** | Prevents privilege escalation via `setuid` binaries. |
| **`privileged: false`**    | Blocks full host access for containers. |
| **`readOnlyRootFilesystem: true`** | Prevents modifications to the filesystem. |
| **`fsGroup`**              | Ensures mounted volumes have proper permissions. |
| **`capabilities.drop`**    | Removes unnecessary Linux capabilities. |
| **`seccompProfile`**       | Restricts system calls. |

---

## Overview Network Policies
Kubernetes **Network Policies** control **ingress (incoming)** and **egress (outgoing)** traffic for pods. They **restrict** communication between pods, namespaces, or external networks.

**Default Behavior:**  
If no NetworkPolicy is applied, all pods **can** communicate freely.

---

## 1. **Key Concepts**
| Term | Description |
|------|-------------|
| **Ingress** | Controls incoming traffic to pods. |
| **Egress** | Controls outgoing traffic from pods. |
| **Pod Selector** | Defines which pods the policy applies to. |
| **Namespace Selector** | Restricts traffic between namespaces. |
| **IP Block** | Restricts access to/from specific IP ranges. |

---

## 2. **Enabling Network Policies**
Network policies require a **CNI (Container Network Interface)** that supports them, such as:
✅ Calico  
✅ Cilium  
✅ WeaveNet  
✅ Kube-router  

To check if your cluster supports network policies:
```sh
kubectl get pods -n kube-system
```
Look for **CNI-related pods** like `calico-node` or `cilium-agent`.

---

## 3. **Deny All Traffic (Default Deny)**
To **isolate** a pod by **blocking all traffic**, use:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {} # Applies to all pods
  policyTypes:
    - Ingress
    - Egress
```

🔹 This policy blocks **all ingress and egress traffic** for pods in the `default` namespace.

---

## 4. **Allow Ingress from Specific Pods**
To allow traffic **only from a specific pod label**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-app
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
✅ **Allows traffic to `backend` pods only from `frontend` pods.**  
❌ All other traffic is blocked.

---

## 5. **Allow Traffic from a Specific Namespace**
To allow ingress **only from a specific namespace**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-namespace
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: frontend
```
✅ **Allows traffic from any pod in a namespace labeled `project=frontend`.**  
❌ Blocks traffic from all other namespaces.

---

## 6. **Allow Traffic from a Specific IP Block**
To allow ingress **only from a specific CIDR (IP range):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ip-range
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 192.168.1.0/24
```
✅ **Allows access only from `192.168.1.0 - 192.168.1.255`.**  
❌ Blocks traffic from all other IPs.

---

## 7. **Allow Only Specific Ports**
To allow traffic **only on a specific port (e.g., 80):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-80
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
```
✅ **Allows traffic to `web` pods from `frontend` pods, only on port 80.**  
❌ Other traffic is denied.

---

## 8. **Egress Rules: Restrict Outbound Traffic**
To allow a pod to communicate **only with external API servers (e.g., 8.8.8.8)**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 8.8.8.8/32
```
✅ **Allows `backend` pods to send requests only to `8.8.8.8`.**  
❌ All other egress traffic is blocked.

---

## 9. **Combining Ingress & Egress**
To allow **both incoming and outgoing** traffic between two services:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-between-apps
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
```
✅ **Allows traffic from `frontend` to `backend` (ingress).**  
✅ **Allows `backend` to send data to `database` (egress).**  
❌ Other traffic is blocked.

---

## 10. **Check Network Policies**
### **List All Policies in a Namespace**
```sh
kubectl get networkpolicy -n default
```

### **Describe a Network Policy**
```sh
kubectl describe networkpolicy allow-from-app -n default
```

### **Check Effective Network Policies on a Pod**
```sh
kubectl get pod mypod -o yaml
```

---

## **Best Practices**
✅ **Apply a `Deny All` policy first and allow only necessary traffic.**  
✅ **Use pod selectors to scope policies instead of broad rules.**  
✅ **Limit external access using IP-based restrictions.**  
✅ **Test policies before applying them in production.**  
✅ **Ensure your CNI supports Network Policies.**  

---

## Summary
| Policy Type | Description |
|-------------|-------------|
| **Deny All Traffic** | Blocks all ingress and egress traffic. |
| **Allow Specific Pods** | Allows traffic between selected pods. |
| **Namespace Restrictions** | Controls inter-namespace communication. |
| **IP-Based Policies** | Allows/denies traffic from specific IP ranges. |
| **Port-Based Policies** | Restricts access to specific ports. |
| **Egress Rules** | Controls outgoing traffic. |

---