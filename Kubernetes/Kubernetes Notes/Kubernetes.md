# Kubernetes Notes

## Basic Kubernetes Commands


### Cluster Info & Context
```sh
kubectl cluster-info  # Get cluster info
```
```sh
kubectl config get-contexts  # List available contexts
```
```sh
kubectl config use-context <context-name>  # Switch context
```

### Pods
```sh
kubectl get pods  # List all pods
```
```sh
kubectl get pods -o wide  # Show pods with extra details
```
```sh
kubectl describe pod <pod-name>  # Get details of a specific pod
```
```sh
kubectl logs <pod-name>  # View logs of a pod
```
```sh
kubectl exec -it <pod-name> -- /bin/sh  # Access a running pod (if it has `sh`)
```
```sh
kubectl delete pod <pod-name>  # Delete a pod
```

### Replicasets
```sh
kubectl get replicasets  # List all replicasets
```
```sh
kubectl describe replicaset <replicaset-name>  # Get details of a replicaset
```
```sh
kubectl delete replicaset <replicaset-name>  # Delete a replicaset
```
```sh
kubectl scale replicaset replicaset --replicas=<number>  # Scale replicaset
```

### Deployments
```sh
kubectl get deployments  # List all deployments
```
```sh
kubectl describe deployment <deployment-name>  # Get details of a deployment
```
```sh
kubectl delete deployment <deployment-name>  # Delete a deployment
```
```sh
kubectl rollout restart deployment <deployment-name>  # Restart a deployment
```
```sh
kubectl scale deployment <deployment-name> --replicas=<number>  # Scale deployment
```

### Services
```sh
kubectl get services  # List all services
```
```sh
kubectl describe service <service-name>  # Get details of a service
```
```sh
kubectl delete service <service-name>  # Delete a service
```

### Namespaces
```sh
kubectl get namespaces  # List all namespaces
```
```sh
kubectl create namespace <namespace>  # Create a new namespace
```
```sh
kubectl delete namespace <namespace>  # Delete a namespace
```

### Config & Secrets
```sh
kubectl get configmaps  # List all config maps
```
```sh
kubectl get secrets  # List all secrets
```
```sh
kubectl describe secret <secret-name>  # Get details of a secret
```

### Nodes
```sh
kubectl get nodes  # List all nodes
```
```sh
kubectl describe node <node-name>  # Get details of a node
```

### Apply & Delete
```sh
kubectl apply -f <file.yaml>  # Apply a configuration file
```
```sh
kubectl delete -f <file.yaml>  # Delete resources from a file
```

### Other Useful Commands
```sh
kubectl get all  # Show all resources in the namespace
```
```sh
kubectl top pod  # Show resource usage of pods
```
```sh
kubectl top node  # Show resource usage of nodes
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

