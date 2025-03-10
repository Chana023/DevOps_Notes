# Kubernetes Scheduling

## Labels and Selectors

### Labels
Labels are key-value pairs assigned to Kubernetes objects to organize and select subsets of objects.

```sh
kubectl label pod <pod-name> env=production  # Add a label to a pod
```
```sh
kubectl label pod <pod-name> env-  # Remove a label from a pod
```
```sh
kubectl get pods --show-labels  # List pods with their labels
```
```sh
kubectl get pods -l env=production  # Get pods with a specific label
```
```sh
kubectl get pods -l 'env in (production, staging)'  # Get pods with multiple label values
```
```sh
kubectl get pods -l 'env notin (development)'  # Get pods that do not have a specific label value
```
```sh
kubectl get pods -l '!env'  # Get pods without the 'env' label
```

### Selectors
Selectors allow filtering of Kubernetes objects based on their labels.

```sh
kubectl get services --selector=app=myapp  # Get services that match a label selector
```
```sh
kubectl get deployments --selector=env=production  # Get deployments with a specific label
```
```sh
kubectl get pods --selector='tier=frontend,app=myapp'  # Select pods with multiple labels
```

#### Editing & Removing Labels
```sh
kubectl label node <node-name> role=worker  # Add a label to a node
```
```sh
kubectl label node <node-name> role-  # Remove a label from a node
```

---

## Taints and Tolerations

### Taints

Add a taint to a node
```sh
kubectl taint nodes node-name key=value:taint-effect
```
- NoSchedule - Pods won't be scheduled on the node unless they tolerate the taint
- PreferNoSchedule - The system will try to avoid placing pods on the node
- NoExecute -  Existing pods will be evicted if they don't tolerate the taint

Remove a taint from a node
```sh
kubectl taint nodes <node-name> <key>[=<value>]:<effect>-
```

### Tolerations

```sh
kubectl taint nodes <node-name> key=value:effect
```

---

## Node Selectors

Limited Usage, probably best to use Node Affinity 
```sh
kubectl label nodes <node-name> <label-key>=<label-value>
```

## Node Afffinity

Node affinity is a way to constrain pods to run on specific nodes based on labels.

### Viewing Node Labels
```sh
kubectl get nodes --show-labels  # List all nodes with their labels
```
```sh
kubectl describe node <node-name>  # Get details of a specific node, including labels
```

### Adding & Removing Node Labels
```sh
kubectl label node <node-name> disktype=ssd  # Add a label to a node
```
```sh
kubectl label node <node-name> disktype-  # Remove a label from a node
```

### Example: Applying Node Affinity in a Pod Spec
In a pod specification (`pod.yaml`), you can define node affinity like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: my-container
    image: nginx
```

## Resource Requirements and Limits

Resource requests and limits help manage CPU and memory usage for containers.

### Viewing Resource Usage
```sh
kubectl top pod  # Show CPU and memory usage of pods
```
```sh
kubectl top node  # Show CPU and memory usage of nodes
```

### Example: Defining Resource Requests & Limits in a Pod Spec
In a pod specification (`pod.yaml`), you can define requests and limits like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    resources:
      requests:
        memory: "128Mi"  # Minimum memory the container requires
        cpu: "250m"  # Minimum CPU the container requires (250 millicores)
      limits:
        memory: "256Mi"  # Maximum memory the container can use
        cpu: "500m"  # Maximum CPU the container can use (500 millicores)
```

#### Checking Resource Requests & Limits
```sh
kubectl get pod my-pod -o yaml | grep resources -A 4  # View resource requests & limits in YAML
```

## Limit Range 

A **LimitRange** object is used to enforce default resource requests and limits within a namespace.

### Viewing LimitRange in a Namespace
```sh
kubectl get limitrange -n <namespace>  # List LimitRange objects in a namespace
```
```sh
kubectl describe limitrange <limitrange-name> -n <namespace>  # View details of a LimitRange
```

### Example: Defining a LimitRange
In a LimitRange specification (`limitrange.yaml`), you can define default CPU and memory limits:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: my-namespace
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"  # Default CPU limit
      memory: "512Mi"  # Default memory limit
    defaultRequest:
      cpu: "250m"  # Default CPU request
      memory: "256Mi"  # Default memory request
    max:
      cpu: "1"  # Max CPU allowed per container
      memory: "1Gi"  # Max memory allowed per container
    min:
      cpu: "100m"  # Min CPU required per container
      memory: "128Mi"  # Min memory required per container
```

#### Checking if Limits Apply to New Pods
```sh
kubectl describe limitrange resource-limits -n my-namespace  # Verify the applied LimitRange
```

#### Deleting the LimitRange
```sh
kubectl delete limitrange resource-limits -n my-namespace  # Remove the LimitRange
```

## Daemon Sets

### What is a DaemonSet?
A **DaemonSet** ensures that a copy of a specific Pod runs on all (or some) nodes in a cluster. It is used for node-level tasks like logging, monitoring, or networking.

### List DaemonSets
```sh
kubectl get daemonsets
kubectl get ds
```

### Describe a DaemonSet
```sh
kubectl describe daemonset <daemonset-name>
```

### Delete a DaemonSet
```sh
kubectl delete daemonset <daemonset-name>
```

---

## Sample DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: example-daemonset
  labels:
    app: example
spec:
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      labels:
        app: example
    spec:
      containers:
        - name: example-container
          image: busybox
          command: ["/bin/sh", "-c", "while true; do echo Hello from DaemonSet; sleep 10; done"]
```

## Important Notes
- A DaemonSet ensures that each node has exactly **one** instance of the Pod.
- If new nodes are added, the DaemonSet automatically schedules Pods on them.
- Deleting a DaemonSet will remove all its Pods.
- Use **nodeSelector** or **affinity** to run Pods on specific nodes.

## Static Pods

## What are Static Pods?
**Static Pods** are managed directly by the **Kubelet**, not the Kubernetes API Server. They are useful for running critical system-level components on specific nodes.

## Key Characteristics
- Defined in a **YAML file** stored on the node (`/etc/kubernetes/manifests/`).
- **Kubelet** ensures they are always running.
- Do **not** use a **ReplicaSet** or **Scheduler**.
- Visible with `kubectl get pods --all-namespaces` but **not** managed via `kubectl apply/delete`.

---

## Important Commands

### Find Static Pod YAML File (on the node)
```sh
ls /etc/kubernetes/manifests/
```

### Delete a Static Pod
```sh
rm /etc/kubernetes/manifests/static-pod.yaml
```
> **Note**: The pod will be removed automatically by Kubelet.

---

### Deploying a Static Pod
1. **Save the YAML file** on the node in `/etc/kubernetes/manifests/`
```sh
sudo vi /etc/kubernetes/manifests/static-pod.yaml
```
2. **Kubelet automatically starts the pod**.

---

## Important Notes
- Used for **critical system components** like control plane services.
- If the file is **deleted**, Kubelet removes the pod.
- No **ReplicaSets, Deployments, or Services** can manage them.
- Logs can be viewed using:
  ```sh
  kubectl logs static-pod-example -n kube-system
  ```

## Kubernetes Scheduler

## What is the Kubernetes Scheduler?
The **Kubernetes Scheduler** is responsible for assigning pods to nodes based on resource availability, constraints, and policies.

## Key Features
- Evaluates **node resources** (CPU, memory, taints, affinity, etc.).
- Uses **scheduling policies** to determine the best node for a pod.
- Can be **customized** or replaced with a custom scheduler.

---

## Important Commands

### View the Current Scheduler
```sh
kubectl get pods -n kube-system | grep kube-scheduler
```

### View Scheduler Logs
```sh
kubectl logs -n kube-system kube-scheduler-<node-name>
```

### Check Scheduler Configuration
```sh
kubectl describe pod kube-scheduler-<node-name> -n kube-system
```

### View Events Related to Scheduling
```sh
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Sample Static Pod YAML for Kubernetes Scheduler

If you are using a **static pod** for the scheduler, the configuration file is typically found in:
```sh
/etc/kubernetes/manifests/kube-scheduler.yaml
```

Example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
    - name: kube-scheduler
      image: k8s.gcr.io/kube-scheduler:v1.28.0
      command:
        - kube-scheduler
        - --config=/etc/kubernetes/scheduler-config.yaml
      volumeMounts:
        - mountPath: /etc/kubernetes
          name: config-volume
  volumes:
    - name: config-volume
      hostPath:
        path: /etc/kubernetes
```

---

## Custom Scheduler Deployment

1. **Create a Custom Scheduler Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduler
  namespace: kube-system
spec:
  containers:
    - name: custom-scheduler
      image: k8s.gcr.io/kube-scheduler:v1.28.0 # Check version here
      command:
        - kube-scheduler
        - --config=/etc/kubernetes/custom-scheduler-config.yaml
```

2. **Apply the Custom Scheduler**
```sh
kubectl apply -f custom-scheduler.yaml
```

3. **Schedule Pods Using the Custom Scheduler**
Modify the pod specification to use the custom scheduler:
```yaml
spec:
  schedulerName: custom-scheduler
```

---

## Important Notes
- The default scheduler binary is **`kube-scheduler`**.
- Custom schedulers can be deployed as a **separate pod**.
- Logs help diagnose why a pod is not getting scheduled.
- To disable the default scheduler, remove `kube-scheduler.yaml` from `/etc/kubernetes/manifests/` (for static pods).

```sh
rm /etc/kubernetes/manifests/kube-scheduler.yaml
```

