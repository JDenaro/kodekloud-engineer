# Day 51: Execute Rolling Updates in Kubernetes

An application currently running on the Kubernetes cluster employs the nginx web server. The Nautilus application development team has introduced some recent changes that need deployment. They've crafted an image `nginx:1.19` with the latest updates.

Execute a rolling update for this application, integrating the `nginx:1.19` image. The deployment is named `nginx-deployment`.
Ensure all pods are operational post-update.
`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Specific Requirements:

1. Execute a rolling update for the application using the `nginx:1.19` image.
2. Update the deployment named `nginx-deployment`.
3. Ensure all pods are operational after the update.

## Solution

Changing the image in a Deployment's Pod template creates a new revision and triggers a rolling update. Kubernetes gradually scales up a new ReplicaSet and scales down the old one, preserving application availability while replacing the Pods.

### ✏️ Step 1: Update the Deployment image

```bash
kubectl edit deployment nginx-deployment
```

In the container specification, change only the image value to:

```yaml
image: nginx:1.19
```

Save and exit with `Esc`, `:wq`, and `Enter`. Kubernetes confirmed the update:

```text
deployment.apps/nginx-deployment edited
```

> **Why:** `kubectl edit deployment nginx-deployment` opens the live Deployment specification in the configured editor. Changing the container image to `nginx:1.19` modifies the Pod template, which creates a new Deployment revision and starts a rolling update without manually deleting the existing Pods.

### 🔄 Step 2: Wait for the rolling update to finish

```bash
kubectl rollout status deployment/nginx-deployment
```

The rollout completed successfully:

```text
deployment "nginx-deployment" successfully rolled out
```

> **Why:** `kubectl rollout status` watches the current rollout of the specified Deployment and waits until the desired replicas are updated and available. The `deployment/nginx-deployment` resource notation identifies both the resource type and its name.

### ✅ Step 3: Verify the Pods and deployed image

```bash
kubectl get pods
kubectl describe deployment nginx-deployment
```

All three replacement Pods were ready and running:

```text
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6655dc8cfb-5jf6j   1/1     Running   0          21s
nginx-deployment-6655dc8cfb-6vnpz   1/1     Running   0          22s
nginx-deployment-6655dc8cfb-djw8w   1/1     Running   0          26s
```

The Deployment details confirmed the new image and replica state:

```text
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
Containers:
  nginx-container:
    Image:         nginx:1.19
OldReplicaSets:  nginx-deployment-fc677cbc9 (0/0 replicas created)
NewReplicaSet:   nginx-deployment-6655dc8cfb (3/3 replicas created)
```

> **Why:** `kubectl get pods` confirms that every replacement Pod is ready and in `Running` state. `kubectl describe deployment nginx-deployment` displays the active container image, rollout strategy, replica availability, and old and new ReplicaSets. The output proves that all three current replicas use the new Pod template while the previous ReplicaSet has been scaled to zero.

## Best Practices

- **Wait for rollout completion.** Use `kubectl rollout status` after changing a Deployment so failed image pulls, readiness failures, or stalled replacements are visible before the task is considered complete.
- **Verify both image and availability.** Running Pods alone do not prove that the intended version was deployed. Check the Deployment image and confirm that updated, available, and desired replica counts agree.
- **Use immutable image references.** A specific version such as `nginx:1.19` is more predictable than `latest`; production environments can use image digests for stronger immutability.
- **Keep rollback history.** Deployment revisions allow a failed release to be rolled back. Review the rollout history and rollback plan before production updates.

### 📚 Official Documentation

- [Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
- [kubectl edit](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [kubectl rollout status](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
