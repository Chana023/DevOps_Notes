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