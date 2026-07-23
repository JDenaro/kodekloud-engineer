# Task 12: Update Deployment and Service in Kubernetes

An application deployed on the Kubernetes cluster requires an update with new features developed by the Nautilus application development team. The existing setup includes a deployment named nginx-deployment and a service named nginx-service. Below are the necessary changes to be implemented without deleting the deployment and service:

1.) Modify the service nodeport from 30008 to 32165

2.) Change the replicas count from 1 to 5

3.) Update the image from nginx:1.17 to nginx:latest

Note: The kubectl utility on the jump-host has been configured to work with the Kubernetes cluster.

## Task Requirements

1. Modify the service `nodePort` from `30008` to `32165`.
2. Change the replicas count from `1` to `5`.
3. Update the image from `nginx:1.17` to `nginx:latest`.
4. Keep the existing Deployment and Service; do not delete them.

## Solution

The existing `nginx-service` and `nginx-deployment` resources were updated in place with `kubectl edit`. This preserves their names, selectors, ClusterIP, and other existing configuration while changing only the requested fields.

### 🛠️ Step 1: Update the Service node port

```bash
kubectl edit service nginx-service
```

In the editor, locate:

```yaml
nodePort: 30008
```

Change it to:

```yaml
nodePort: 32165
```

Save and exit with `Esc`, `:wq`, and `Enter`.

The Service was updated successfully:

```text
thor@jump-host ~$ kubectl edit service nginx-service
service/nginx-service edited
thor@jump-host ~$
```

> **Why:** `kubectl edit` opens the live resource definition in the configured text editor. `service` selects the Kubernetes Service resource type, and `nginx-service` identifies the existing Service. `nodePort` is the port exposed on each cluster node, so changing it from `30008` to `32165` changes the external node port without recreating the Service.

### ⚙️ Step 2: Update the Deployment replicas and image

```bash
kubectl edit deployment nginx-deployment
```

In the editor, change:

```yaml
replicas: 1
```

to:

```yaml
replicas: 5
```

Also change:

```yaml
image: nginx:1.17
```

to:

```yaml
image: nginx:latest
```

Save and exit with `Esc`, `:wq`, and `Enter`.

The Deployment was updated successfully:

```text
thor@jump-host ~$ kubectl edit deployment nginx-deployment
deployment.apps/nginx-deployment edited
thor@jump-host ~$
```

> **Why:** `deployment` selects the Kubernetes Deployment resource type, and `nginx-deployment` identifies the existing Deployment. The `replicas` field tells the Deployment controller to maintain five Pods instead of one. Updating the container image to `nginx:latest` makes the Deployment replace the old Pods with Pods that use the requested image. Because the resource was edited in place, the existing Deployment remains intact and Kubernetes performs the change through its controller.

### ✅ Step 3: Verify the updated resources

The lab was reported successful after the two edit operations. To inspect the final values manually, use:

```bash
kubectl get deployment nginx-deployment
kubectl get service nginx-service
```

The expected state is a Deployment configured for five replicas and a Service using node port `32165`. The Deployment's Pod template should reference `nginx:latest`.

> **Why:** `get` retrieves the current state of Kubernetes resources. The first command checks the Deployment's replica status, while the second displays the Service ports, including `nodePort`. A separate `describe` or `get` output was not captured because the lab was accepted immediately after the requested edits.

## Best Practices

- **Edit existing resources in place.** This preserves resource identity and avoids unnecessary downtime caused by deleting and recreating the Deployment or Service.
- **Change only the requested fields.** Leave selectors, ports, ClusterIP, labels, and container settings unchanged unless the task requires them.
- **Use a valid node port.** The requested `32165` is within Kubernetes' default NodePort range and must not already be assigned to another Service.
- **Let the Deployment controller manage replicas.** Changing `replicas` allows Kubernetes to create or remove Pods until the desired count is reached.
- **Pin production images when possible.** The lab requires `nginx:latest`, but a version tag or immutable image digest is more predictable for production workloads.
- **Check rollout health after production changes.** For real deployments, use `kubectl rollout status deployment/nginx-deployment` and inspect the Pods after changing the image or replica count.

### 📚 Official Documentation

- [`kubectl edit` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [`kubectl get` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes rolling updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)
