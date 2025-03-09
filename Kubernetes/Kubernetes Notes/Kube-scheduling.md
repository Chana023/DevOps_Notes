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