# Day 48: Deploy Pods in Kubernetes Cluster

The Nautilus DevOps team is diving into Kubernetes for application management. One team member has a task to create a pod according to the details below:

1. Create a pod named `pod-httpd` using the `httpd` image with the `latest` tag. Ensure to specify the tag as `httpd:latest`.
2. Set the `app` label to `httpd_app`, and name the container as `httpd-container`.

`Note`: The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Specific Requirements:

1. Create a pod named `pod-httpd` using the `httpd` image with the `latest` tag. Ensure to specify the tag as `httpd:latest`.
2. Set the `app` label to `httpd_app`, and name the container as `httpd-container`.

## Solution

A Pod manifest declares the required Kubernetes object, its label, and its container configuration in one versionable file. The preconfigured `kubectl` context on the jump host applies that manifest to the cluster.

### 📝 Step 1: Create the Pod manifest

```bash
vi pod-httpd.yaml
```

Add the following content, then save and exit with `Esc`, `:wq`, and `Enter`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd
  labels:
    app: httpd_app
spec:
  containers:
    - name: httpd-container
      image: httpd:latest
```

> **Why:** `vi pod-httpd.yaml` creates a declarative Kubernetes manifest. `apiVersion: v1` selects the core Kubernetes API, while `kind: Pod` declares the smallest deployable Kubernetes workload. `metadata.name` assigns the required Pod name and `metadata.labels.app` assigns `app=httpd_app`, which can later be used by selectors such as Services. `spec.containers` defines the Pod's containers; `name` assigns `httpd-container` and `image: httpd:latest` explicitly selects the required Apache HTTP Server image and tag.

### 🚀 Step 2: Create the Pod in the cluster

```bash
kubectl apply -f pod-httpd.yaml
```

Kubernetes confirmed the resource creation:

```text
pod/pod-httpd created
```

> **Why:** `kubectl apply` sends a declarative configuration to the Kubernetes API server. `-f pod-httpd.yaml` selects the manifest file that contains the Pod definition. Kubernetes stored the object, then the scheduler assigned it to a node and the node's container runtime pulled and started `httpd:latest`.

### ✅ Step 3: Verify the Pod is running

```bash
kubectl get pods
```

The cluster reported the required Pod as ready and running:

```text
NAME        READY   STATUS    RESTARTS   AGE
pod-httpd   1/1     Running   0          19s
```

> **Why:** `kubectl get pods` lists Pods in the current namespace. `1/1` in `READY` confirms that the only declared container is ready, and `Running` confirms that Kubernetes successfully started the Pod.

## Best Practices

- **Prefer manifests over imperative shortcuts.** A manifest makes the Pod name, label, image, and container name explicit, reviewable, and reproducible.
- **Avoid `latest` in production.** This lab explicitly requires `httpd:latest`, but production workloads should use an immutable image version or digest so deployments do not change unexpectedly.
- **Use a controller for long-lived workloads.** A standalone Pod does not provide replica management or a rollout strategy. Use a Deployment when an application needs availability, updates, or multiple replicas.
- **Apply labels consistently.** Labels are the mechanism Services, NetworkPolicies, and operational tooling use to select workloads. Choose a stable labeling convention as the environment grows.

### 📚 Official Documentation

- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl get](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
