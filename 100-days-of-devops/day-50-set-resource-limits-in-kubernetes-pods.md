# Day 50: Set Resource Limits in Kubernetes Pods

The Nautilus DevOps team has noticed performance issues in some Kubernetes-hosted applications due to resource constraints. To address this, they plan to set limits on resource utilization. Here are the details:

Create a pod named `httpd-pod` with a container named `httpd-container`. Use the `httpd` image with the `latest` tag (specify as `httpd:latest`). Configure the following container-level resource requests and limits for the container:
Requests: Memory: `15Mi`, CPU: `100m`
Limits: Memory: `20Mi`, CPU: `100m`
`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Specific Requirements:

1. Create a pod named `httpd-pod` with a container named `httpd-container` using the image `httpd:latest`.
2. Configure container-level resource requests of memory `15Mi` and CPU `100m`.
3. Configure container-level resource limits of memory `20Mi` and CPU `100m`.

## Solution

Kubernetes resource requests reserve capacity used by the scheduler to place a Pod, while limits define the maximum resources the container may use. A Pod manifest declares these values precisely in the container specification.

### 📝 Step 1: Create the resource-managed Pod manifest

```bash
vi httpd-pod.yaml
```

Add the following content, then save and exit with `Esc`, `:wq`, and `Enter`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod
spec:
  containers:
    - name: httpd-container
      image: httpd:latest
      resources:
        requests:
          memory: "15Mi"
          cpu: "100m"
        limits:
          memory: "20Mi"
          cpu: "100m"
```

> **Why:** `vi httpd-pod.yaml` creates a declarative manifest for the required Pod. `metadata.name` sets `httpd-pod`, and the container entry sets the required name and `httpd:latest` image. The `resources` field applies settings to this container: `requests` tells the scheduler the minimum CPU and memory capacity it must find on a node, while `limits` establishes the maximum allowed usage. `100m` means 100 millicores, or 0.1 CPU core. `Mi` means mebibytes, a binary memory unit.

### 🚀 Step 2: Create the Pod in the cluster

```bash
kubectl apply -f httpd-pod.yaml
```

Kubernetes confirmed the resource creation:

```text
pod/httpd-pod created
```

> **Why:** `kubectl apply` submits the declarative configuration to the Kubernetes API server. `-f httpd-pod.yaml` selects the manifest file. The scheduler uses the requested CPU and memory values when choosing a node for the Pod.

### ✅ Step 3: Verify the Pod state and configured resources

```bash
kubectl get pod httpd-pod
kubectl describe pod httpd-pod
```

The Pod became ready and running:

```text
NAME        READY   STATUS    RESTARTS   AGE
httpd-pod   1/1     Running   0          45s
```

`kubectl describe` confirmed the required values:

```text
Limits:
  cpu:     100m
  memory:  20Mi
Requests:
  cpu:        100m
  memory:     15Mi
QoS Class:                   Burstable
```

> **Why:** `kubectl get pod httpd-pod` confirms that the requested Pod is running and that its only container is ready. `kubectl describe pod httpd-pod` displays the detailed Pod specification and runtime information, including the container-level resource values. Kubernetes assigned `Burstable` Quality of Service because the memory request is lower than its memory limit.

## Best Practices

- **Set requests and limits together.** Requests let the scheduler make informed placement decisions, while limits prevent a single workload from consuming unbounded resources.
- **Size values from observed usage.** The values in this lab are fixed requirements. In production, use application metrics and load testing to choose requests and limits that protect both the workload and the cluster.
- **Avoid `latest` in production.** This lab explicitly requires `httpd:latest`, but production workloads should reference a specific version or image digest for predictable deployments.
- **Use controllers for durable workloads.** A standalone Pod does not reschedule replicas or provide rolling updates. Use a Deployment for a long-lived stateless application.

### 📚 Official Documentation

- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [kubectl apply](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
