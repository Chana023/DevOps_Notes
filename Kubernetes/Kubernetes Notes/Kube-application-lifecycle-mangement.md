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