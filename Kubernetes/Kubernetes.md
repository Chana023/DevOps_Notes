# Kubernetes Notes

## Basic Kubernetes Commands

### Cluster Info & Context
- `kubectl cluster-info` – Get cluster info  
- `kubectl config get-contexts` – List available contexts  
- `kubectl config use-context <context-name>` – Switch context  

### Pods
- `kubectl get pods` – List all pods  
- `kubectl get pods -o wide` – Show pods with extra details  
- `kubectl describe pod <pod-name>` – Get details of a specific pod  
- `kubectl logs <pod-name>` – View logs of a pod  
- `kubectl exec -it <pod-name> -- /bin/sh` – Access a running pod (if it has `sh`)  
- `kubectl delete pod <pod-name>` – Delete a pod  

### Replicasets

- `kubectl get replicasets` – List all replicasets  
- `kubectl describe replicaset <replicaset-name>` – Get details of a replicaset  
- `kubectl delete replicaset <replicaset-name>` – Delete a replicaset  
- `kubectl scale replicaset replicaset --replicas=<number>` – Scale replicaset


### Deployments
- `kubectl get deployments` – List all deployments  
- `kubectl describe deployment <deployment-name>` – Get details of a deployment  
- `kubectl delete deployment <deployment-name>` – Delete a deployment  
- `kubectl rollout restart deployment <deployment-name>` – Restart a deployment  
- `kubectl scale deployment <deployment-name> --replicas=<number>` – Scale deployment  

### Services
- `kubectl get services` – List all services  
- `kubectl describe service <service-name>` – Get details of a service  
- `kubectl delete service <service-name>` – Delete a service  

### Namespaces
- `kubectl get namespaces` – List all namespaces  
- `kubectl create namespace <namespace>` – Create a new namespace  
- `kubectl delete namespace <namespace>` – Delete a namespace  

### Config & Secrets
- `kubectl get configmaps` – List all config maps  
- `kubectl get secrets` – List all secrets  
- `kubectl describe secret <secret-name>` – Get details of a secret  

### Nodes
- `kubectl get nodes` – List all nodes  
- `kubectl describe node <node-name>` – Get details of a node  

### Apply & Delete
- `kubectl apply -f <file.yaml>` – Apply a configuration file  
- `kubectl delete -f <file.yaml>` – Delete resources from a file  

### Other Useful Commands
- `kubectl get all` – Show all resources in the namespace  
- `kubectl top pod` – Show resource usage of pods  
- `kubectl top node` – Show resource usage of nodes  

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


### Useful Options
- `--dry-run=client`: Simulates resource creation without actually creating it.
- `-o yaml`: Outputs the resource definition in YAML format.
- `--selector` or `-l`: allows you to select based on labels placed on pods

Combining these options allows quick generation of resource definition files for modification before actual creation.

---
# Imperative commands (For fast templating and testing only)

## Pod

### Create an NGINX Pod
```sh
kubectl run nginx --image=nginx
```

### Generate Pod Manifest YAML (without creating it)
```sh
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

## Deployment

### Create a Deployment
```sh
kubectl create deployment --image=nginx nginx
```

### Generate Deployment YAML (without creating it)
```sh
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml
```

### Generate Deployment with 4 Replicas
```sh
kubectl create deployment nginx --image=nginx --replicas=4
```

### Scale an Existing Deployment
```sh
kubectl scale deployment nginx --replicas=4
```

### Save Deployment YAML for Modification
```sh
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml
```

Modify `nginx-deployment.yaml` to set replicas or other parameters before creating the deployment.

## Service

### Create a ClusterIP Service for Redis (Port 6379)
```sh
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```

Alternative:
```sh
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml
```
*Note: This assumes selectors as `app=redis`, so modify selectors if needed.*

### Create a NodePort Service for NGINX (Port 80 → NodePort 30080)

```sh
kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml
```
*Note: This automatically selects labels but does not allow setting a node port manually.*

Alternative:
```sh
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```
*Note: This does not use pod labels as selectors.*

### Recommendation
Use `kubectl expose` to generate a service definition and manually specify the `nodePort` before applying it.

