# Day 49: Deploy Applications with Kubernetes Deployments

The Nautilus DevOps team is delving into Kubernetes for app management. One team member needs to create a deployment following these details:

Create a deployment named `nginx` to deploy the application `nginx` using the image `nginx:latest` (ensure to specify the tag)
`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Specific Requirements:

1. Create a deployment named `nginx` to deploy the application `nginx` using the image `nginx:latest` (ensure to specify the tag).

## Solution

The `kubectl create deployment` command creates a Deployment controller that manages the application's Pods. The jump host already has access to the cluster, so the Deployment can be created directly from its configured Kubernetes context.

### 🚀 Step 1: Create the Nginx Deployment

```bash
kubectl create deployment nginx --image=nginx:latest
```

Kubernetes confirmed the resource creation:

```text
deployment.apps/nginx created
```

> **Why:** `kubectl create deployment` creates a Deployment resource in the current namespace. `nginx` is the required Deployment name. `--image=nginx:latest` sets the container image template managed by the Deployment and explicitly specifies the required `latest` tag. The Deployment controller creates and maintains the Pod replica for that template.

### ✅ Step 2: Verify the Deployment is available

```bash
kubectl get deployment
```

The cluster reported the Deployment as ready:

```text
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           16s
```

> **Why:** `kubectl get deployment` lists Deployments in the current namespace. `READY 1/1` confirms that the desired replica is ready. `UP-TO-DATE 1` confirms that the replica uses the current Deployment template, and `AVAILABLE 1` confirms that Kubernetes considers it available to serve workload traffic.

## Best Practices

- **Use Deployments for stateless applications.** Unlike a standalone Pod, a Deployment uses a ReplicaSet to maintain the desired number of Pods and supports controlled rollouts and rollbacks.
- **Avoid `latest` in production.** The lab explicitly requires `nginx:latest`, but production deployments should use an immutable image version or digest so a rollout always uses the intended artifact.
- **Use declarative manifests for managed environments.** `kubectl create` is efficient for this lab; version-controlled Deployment manifests make production configuration reviewable and repeatable.
- **Define health checks and resource requests.** Production Deployments should include readiness and liveness probes plus CPU and memory requests and limits so Kubernetes can route traffic and schedule workloads reliably.

### 📚 Official Documentation

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [kubectl create deployment](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_deployment/)
- [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
