# Task 05: Execute Rolling Updates in Kubernetes

An application currently running on the Kubernetes cluster employs the nginx web server. The Nautilus application development team has introduced some recent changes that need deployment. They've crafted an image nginx:1.17 with the latest updates.

Execute a rolling update for this application, integrating the nginx:1.17 image. The deployment is named nginx-deployment.

Ensure all pods are operational post-update.

Note: The kubectl utility on the jump-host has been configured to work with the Kubernetes cluster.

## Task Requirements

1. Execute a rolling update for the application, integrating the `nginx:1.17` image.
2. Update the Deployment named `nginx-deployment`.
3. Ensure all Pods are operational after the update.

## Solution

A rolling update changes the Pod template used by an existing Deployment. Kubernetes gradually replaces Pods using the old image with Pods using the new image, allowing the application to remain available during the change.

There are two straightforward ways to update the image. `kubectl set image` changes the image directly and is the fastest option. `kubectl edit` opens the live Deployment definition so the image can be changed manually. Both methods trigger a rolling update; only one method is needed for the task.

## Imperative Method

### 🚀 Step 1: Update the image with `kubectl set image`

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.17
```

The expected output is:

```text
deployment.apps/nginx-deployment image updated
```

> **Why:** `kubectl set image` updates the image in an existing Pod template. `deployment/nginx-deployment` identifies the Deployment resource, `nginx` on the left side identifies the existing container, and `nginx:1.17` on the right side is the new image and explicit tag. Changing the Deployment's Pod template causes Kubernetes to create replacement Pods and gradually remove the old ones.

## Edit Method

### ✏️ Step 1: Edit the existing Deployment

```bash
kubectl edit deployment nginx-deployment
```

When the editor opens, find the container's current `image:` field and change its value to:

```yaml
image: nginx:1.17
```

Save and exit the editor with `Esc`, `:wq`, and `Enter`. The command used in the successful lab was:

```text
thor@jump-host ~$ kubectl edit deployment nginx-deployment
deployment.apps/nginx-deployment edited
thor@jump-host ~$
```

> **Why:** `edit` retrieves the live Deployment from the Kubernetes API and opens it in the configured editor. `deployment` identifies the resource type, and `nginx-deployment` identifies the resource to edit. Updating only the container image in the Pod template is enough to trigger a rolling update. Avoid changing the Deployment name, selector, or unrelated fields while editing the live resource.

### ✅ Step 2: Wait for the rolling update to complete

```bash
kubectl rollout status deployment/nginx-deployment
```

The successful lab output was:

```text
thor@jump-host ~$ kubectl rollout status deployment/nginx-deployment
deployment "nginx-deployment" successfully rolled out
thor@jump-host ~$
```

> **Why:** `rollout status` watches the latest Deployment rollout until it finishes. `deployment/nginx-deployment` identifies the resource whose update is being monitored. The message `successfully rolled out` confirms that Kubernetes completed the replacement process and that the Deployment's Pods are operational according to its rollout conditions.

## Best Practices

- **Update the Deployment template instead of editing individual Pods.** Pods managed by a Deployment are replaceable; changing the template lets the Deployment controller create the correct replacements.
- **Use a specific image tag for controlled releases.** `nginx:1.17` identifies the intended application version instead of relying on an ambiguous tag.
- **Wait for rollout completion.** `kubectl rollout status` confirms that the new Pods became ready and the rolling update finished successfully.
- **Prefer `kubectl set image` for a single image change.** It reduces the chance of accidentally modifying unrelated Deployment fields.
- **Use `kubectl edit` carefully.** It is useful for live troubleshooting, but manual edits can introduce formatting or configuration mistakes if unrelated fields are changed.
- **Keep rollback options in mind.** Deployments retain rollout history, so a failed or problematic update can be investigated and rolled back with the appropriate Kubernetes rollout commands.

### 📚 Official Documentation

- [`kubectl set image` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_image/)
- [`kubectl edit` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/)
- [`kubectl rollout status` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_status/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
