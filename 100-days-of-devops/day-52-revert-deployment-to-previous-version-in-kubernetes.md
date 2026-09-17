# Day 52: Revert Deployment to Previous Version in Kubernetes

Earlier today, the Nautilus DevOps team deployed a new release for an application. However, a customer has reported a bug related to this recent release. Consequently, the team aims to revert to the previous version.

There exists a deployment named `nginx-deployment`; initiate a rollback to the previous revision.
`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Specific Requirements:

1. There exists a deployment named `nginx-deployment`; initiate a rollback to the previous revision.

## Solution

Kubernetes Deployments preserve rollout revisions so an earlier Pod template can be restored. Running `kubectl rollout undo` without selecting a revision rolls the Deployment back to its immediately previous revision while keeping the rollback itself in the revision history.

### 📜 Step 1: Review the Deployment history

```bash
kubectl rollout history deployment/nginx-deployment
```

The Deployment had two revisions before the rollback:

```text
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment nginx-deployment nginx-container=nginx:alpine-perl --record=true
```

> **Why:** `kubectl rollout history` displays the stored revisions for a workload. `deployment/nginx-deployment` identifies the Deployment named `nginx-deployment`. The output showed revision `2` as the current update and revision `1` as the previous version that Kubernetes could restore.

### ↩️ Step 2: Roll back to the previous revision

```bash
kubectl rollout undo deployment/nginx-deployment
```

Kubernetes accepted the rollback:

```text
deployment.apps/nginx-deployment rolled back
```

> **Why:** `kubectl rollout undo` restores the previous Pod template for the specified Deployment. Because no `--to-revision` option was supplied, Kubernetes selected the immediately preceding revision automatically and started a rolling replacement of the Pods.

### 🔄 Step 3: Wait for the rollback rollout to finish

```bash
kubectl rollout status deployment/nginx-deployment
```

The rollout completed successfully:

```text
deployment "nginx-deployment" successfully rolled out
```

> **Why:** `kubectl rollout status` watches the active rollout and waits until all desired replicas use the restored Pod template and are available. A successful result confirms that the rollback did not stall.

### ✅ Step 4: Verify the Pods and restored version

```bash
kubectl get pods
kubectl describe deployment nginx-deployment
```

All three replacement Pods were ready and running:

```text
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-fc677cbc9-7bhcw   1/1     Running   0          19s
nginx-deployment-fc677cbc9-pfnmc   1/1     Running   0          20s
nginx-deployment-fc677cbc9-vcvjk   1/1     Running   0          18s
```

The Deployment details confirmed the restored image and healthy replicas:

```text
Annotations:            deployment.kubernetes.io/revision: 3
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
Containers:
  nginx-container:
    Image:         nginx:1.16
OldReplicaSets:  nginx-deployment-55658c8544 (0/0 replicas created)
NewReplicaSet:   nginx-deployment-fc677cbc9 (3/3 replicas created)
```

> **Why:** `kubectl get pods` confirms that every restored replica is ready and running. `kubectl describe deployment nginx-deployment` shows the active image, replica counts, and ReplicaSets. The restored template uses `nginx:1.16`, all three replicas are available, and the failed release's ReplicaSet is scaled to zero. Kubernetes recorded the rollback as revision `3`; a rollback restores an earlier template but still creates a new point in the Deployment's history.

## Best Practices

- **Review history before rollback.** Confirm that the Deployment has a usable prior revision and understand which release is being replaced.
- **Wait for the rollback to complete.** A rollback command being accepted does not prove that the replacement Pods became healthy; always check the rollout status.
- **Verify the restored image and availability.** Confirm both the application version and the desired, updated, available, and unavailable replica counts.
- **Keep revision history meaningful.** Use clear deployment change annotations or external release records so operators can identify the purpose of each revision during an incident.

### 📚 Official Documentation

- [Rolling Back a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
- [kubectl rollout history](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_history/)
- [kubectl rollout undo](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_undo/)
- [kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
