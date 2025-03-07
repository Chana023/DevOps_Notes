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

---
## Shortcut or Do it faster commands 

- `kubectl run nginx --image=nginx` - Create an NGINX Pod

- `kubectl run nginx --image=nginx --dry-run=client -o yaml` - Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)

- `kubectl create deployment --image=nginx nginx` - Create a deployment

- `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml` - Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)

- `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml` - Generate Deployment YAML file (-o yaml). Don’t create it(–dry-run) and save it to a file.

- `kubectl create -f nginx-deployment.yaml` - Make necessary changes to the file (for example, adding more replicas) and then create the deployment.