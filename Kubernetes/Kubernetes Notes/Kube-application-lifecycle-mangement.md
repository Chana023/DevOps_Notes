# Application Life Cycle Management

## Rolling Updates and Rollbacks

## What are Rollouts and Rollbacks?
- **Rollout**: Deploys a new version of an application with a controlled update strategy.
- **Rollback**: Reverts to a previous deployment version if issues occur.

---

## Key Commands

### Check Rollout Status
```sh
kubectl rollout status deployment <deployment-name>
```

### View Rollout History
```sh
kubectl rollout history deployment <deployment-name>
```

### Pause a Rollout
```sh
kubectl rollout pause deployment <deployment-name>
```

### Resume a Paused Rollout
```sh
kubectl rollout resume deployment <deployment-name>
```

### Undo a Rollout (Rollback)
```sh
kubectl rollout undo deployment <deployment-name>
```

### Rollback to a Specific Revision
```sh
kubectl rollout undo deployment <deployment-name> --to-revision=<revision-number>
```

---

## Sample Deployment with Rollout Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
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
          image: nginx:latest
```

---

## Rollout Strategies
1. **Rolling Update (default)**
   - Updates pods gradually, minimizing downtime.
   - Configured via `maxUnavailable` and `maxSurge`.

2. **Recreate**
   - Deletes all old pods before creating new ones.
   - Causes downtime but ensures a clean state.

```yaml
spec:
  strategy:
    type: Recreate
```

---

## Important Notes
- **Rolling updates** allow smooth deployments without downtime.
- **Rollback quickly** if a deployment fails.
- **Check history** before rolling back to ensure the correct version.
- **Use `pause` and `resume`** for controlled releases.

## Environment Variables

### Overview
Kubernetes allows you to define environment variables for containers using:
1. **Static values** (`env`)
2. **ConfigMaps** (`envFrom`)
3. **Secrets** (`envFrom`)
4. **Downward API** (pod metadata)

---

### 1. Defining Environment Variables in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-demo
spec:
  containers:
    - name: demo-container
      image: busybox
      command: ["/bin/sh", "-c", "echo $MY_VAR && sleep 3600"]
      env:
        - name: MY_VAR
          value: "Hello, Kubernetes!"
```
- The container will have `MY_VAR=Hello, Kubernetes!`

---

### 2. Using ConfigMaps for Environment Variables

#### Creating a ConfigMap from a Literal
```sh
kubectl create configmap my-config --from-literal=CONFIG_VALUE=production
```

#### Creating a ConfigMap from a File
1. Create a file (`config.env`):
   ```
   CONFIG_VALUE=production
   LOG_LEVEL=debug
   ```
2. Run:
   ```sh
   kubectl create configmap my-config --from-env-file=config.env
   ```

#### Define a ConfigMap in YAML
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  CONFIG_VALUE: "production"
  LOG_LEVEL: "debug"
```

#### Use ConfigMap in a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo
spec:
  containers:
    - name: demo-container
      image: busybox
      command: ["/bin/sh", "-c", "echo $CONFIG_VALUE && sleep 3600"]
      envFrom:
        - configMapRef:
            name: my-config
```
- All key-value pairs in `my-config` are loaded as environment variables.

---

### 3. Using Secrets for Environment Variables

#### Creating a Secret from a Literal
```sh
kubectl create secret generic my-secret --from-literal=SECRET_KEY=mysecretvalue
```

#### Creating a Secret from a File
1. Create a file (`secret.env`):
   ```
   SECRET_KEY=mysecretvalue
   API_TOKEN=abcdef12345
   ```
2. Run:
   ```sh
   kubectl create secret generic my-secret --from-env-file=secret.env
   ```

#### Define a Secret in YAML
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  SECRET_KEY: bXlzZWNyZXR2YWx1ZQ==  # Base64 encoded "mysecretvalue"
  API_TOKEN: YWJjZGVmMTIzNDU=       # Base64 encoded "abcdef12345"
```
> **Note:** Secret values must be **Base64-encoded**.

#### Use Secret in a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
    - name: demo-container
      image: busybox
      command: ["/bin/sh", "-c", "echo $SECRET_KEY && sleep 3600"]
      envFrom:
        - secretRef:
            name: my-secret
```
- **Secrets are more secure** than ConfigMaps.

---

### 4. Using the Downward API (Pod Metadata)

#### Example: Access Pod Name as an Environment Variable
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-demo
spec:
  containers:
    - name: demo-container
      image: busybox
      command: ["/bin/sh", "-c", "echo $POD_NAME && sleep 3600"]
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```
- The container gets its **own Pod name** as `POD_NAME`.

---

### 5. Passing Environment Variables via `kubectl run`
```sh
kubectl run env-demo --image=busybox --env="MY_VAR=Hello" --command -- sleep 3600
```
- Creates a pod with `MY_VAR=Hello`.

---

### 6. Viewing Environment Variables in a Running Pod
```sh
kubectl exec -it <pod-name> -- printenv
```

---

### Important Notes
- **Secrets are more secure** than ConfigMaps.
- **Downward API** is useful for injecting pod metadata dynamically.
- **Use `envFrom` for bulk loading** from ConfigMaps/Secrets, and `env` for specific variables.
- **Avoid hardcoding sensitive values**—use Secrets instead.
