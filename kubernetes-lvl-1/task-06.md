# Task 06: Revert Deployment to Previous Version in Kubernetes

Earlier today, the Nautilus DevOps team deployed a new release for an application. However, a customer has reported a bug related to this recent release. Consequently, the team aims to revert to the previous version.

There exists a deployment named nginx-deployment; initiate a rollback to the previous revision.

Note: The kubectl utility on the jump-host has been configured to work with the Kubernetes cluster.

## Task Requirements

1. Initiate a rollback to the previous revision of the existing Deployment named `nginx-deployment`.

## Solution

Kubernetes Deployments keep rollout history when their Pod templates change. The `kubectl rollout undo` command uses that history to restore the previous revision. Because the task requests the immediately previous revision, no specific revision number is needed.

After starting the rollback, `kubectl rollout status` waits while Kubernetes replaces the newer Pods with Pods from the previous revision. The command only finishes successfully after the Deployment completes the rollback.

### ↩️ Step 1: Roll back to the previous revision

```bash
kubectl rollout undo deployment nginx-deployment
```

The lab command successfully rolled back the Deployment:

```text
thor@jump-host ~$ kubectl rollout undo deployment nginx-deployment
deployment.apps/nginx-deployment rolled back
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client. `rollout` manages the revision history and rollout operations of workload resources, while `undo` reverts a resource to a previous rollout. `deployment` identifies the resource type, and `nginx-deployment` identifies the Deployment to restore. Because no `--to-revision` option is supplied, Kubernetes uses the previous revision by default. The output confirms that the rollback request was accepted.

### ✅ Step 2: Wait for the rollback to complete

```bash
kubectl rollout status deployment/nginx-deployment
```

The command watched the rollback until all replacement Pods became ready:

```text
thor@jump-host ~$ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
thor@jump-host ~$
```

> **Why:** `rollout status` watches the latest rollout until it finishes. The resource notation `deployment/nginx-deployment` combines the resource type and name with a slash. The progress messages show Kubernetes updating the three replicas and terminating the remaining old replica. The final `successfully rolled out` message confirms that the rollback completed successfully and the Deployment returned to the previous revision.

## Best Practices

- **Use the Deployment rollback mechanism.** Reverting the Deployment revision lets Kubernetes restore the complete previous Pod template instead of changing individual Pods manually.
- **Wait for rollout completion.** A rollback request being accepted does not mean that every Pod has already been replaced; `kubectl rollout status` confirms the final state.
- **Use revision history intentionally.** The default rollback targets the immediately previous revision, while `--to-revision` can be used when a specific older revision is required.
- **Investigate the failed release after stabilizing the service.** A rollback restores availability, but the underlying bug should still be analyzed before attempting another deployment.
- **Keep Deployment updates reversible.** Use controlled image updates and maintain rollout history so a problematic release can be reverted safely.

### 📚 Official Documentation

- [`kubectl rollout undo` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_undo/)
- [`kubectl rollout status` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
- [`kubectl rollout` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
