# Task 04: Set Resource Limits in Kubernetes Pods

The Nautilus DevOps team has noticed performance issues in some Kubernetes-hosted applications due to resource constraints. To address this, they plan to set limits on resource utilization. Here are the details:

## Task Requirements

1. Create a pod named `httpd-pod` with a container named `httpd-container`. Use the `httpd` image with the latest tag (specify as `httpd:latest`). Configure the following container-level resource requests and limits for the container:

   Requests: Memory: `15Mi`, CPU: `100m`

   Limits: Memory: `20Mi`, CPU: `100m`

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

Resource settings must be placed inside the container definition because the task requests container-level values. The `requests` values tell the Kubernetes scheduler the amount of CPU and memory needed for placement. The `limits` values define the maximum resources that the container is allowed to use.

The Pod uses the exact names and image requested by the task: `httpd-pod`, `httpd-container`, and `httpd:latest`.

### 📝 Step 1: Create the Pod manifest

```bash
vi httpd-pod.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

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

> **Why:** `vi` is a terminal text editor. The `i` key enters insert mode, `Esc` leaves insert mode, and `:wq` saves the file and exits. `apiVersion: v1` selects the core Kubernetes API, `kind: Pod` identifies the resource type, and `metadata.name` assigns the required Pod name. Under `spec.containers`, `name` sets the required container name and `image` selects `httpd:latest`, including the explicit `latest` tag. The `resources` section contains the container's resource settings. `requests` describe the resources needed for scheduling, while `limits` define the maximum allowed usage.

The memory values use `Mi`, which represents mebibytes. The CPU values use `m`, which represents millicores; `100m` is one tenth of a CPU core. The memory limit is higher than the memory request, while both CPU values are exactly `100m` as required by the task.

### 🚀 Step 2: Create the Pod with the resource settings

```bash
kubectl apply -f httpd-pod.yaml
```

The lab completed successfully:

```text
thor@jump-host ~$ vi httpd-pod.yaml
thor@jump-host ~$ kubectl apply -f httpd-pod.yaml
pod/httpd-pod created
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client, and `apply` sends the desired resource configuration to the cluster. The `-f` option tells `kubectl` to read the definition from a file, and `httpd-pod.yaml` is the manifest created in the previous step. The output `pod/httpd-pod created` confirms that Kubernetes accepted the Pod definition.

### ✅ Step 3: Verify the Pod resource configuration

The lab validator accepted the Pod after the `kubectl apply` command, so this additional check was not required during the session. If a manual check is needed, run:

```bash
kubectl describe pod httpd-pod
```

The output should show the requested container image and resource values in the container details:

```text
Name:         httpd-pod
Containers:
  httpd-container:
    Image:      httpd:latest
    Limits:
      cpu:      100m
      memory:   20Mi
    Requests:
      cpu:      100m
      memory:   15Mi
```

> **Why:** `describe` displays detailed information about a Kubernetes resource. `pod` selects the resource type, and `httpd-pod` selects the Pod by name. The container details allow the image, container name, requests, and limits to be checked together.

## Best Practices

- **Set requests for predictable scheduling.** Kubernetes uses CPU and memory requests when deciding whether a node has enough available capacity for a Pod.
- **Set limits to control maximum usage.** Limits help prevent one container from consuming an uncontrolled amount of a node's CPU or memory.
- **Place resources at the container level when required.** The `resources` block belongs under the specific container when the task asks for container-level settings.
- **Use valid resource units.** `Mi` represents mebibytes, while `m` represents millicores. The uppercase and lowercase forms have different meanings.
- **Keep memory requests at or below limits.** A request of `15Mi` and a limit of `20Mi` gives the container a guaranteed scheduling requirement and a defined maximum.
- **Use realistic production values.** The small values in this lab are intentional; production values should be based on observed application usage and tested capacity planning.

### 📚 Official Documentation

- [Kubernetes resource management for Pods and containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes container images](https://kubernetes.io/docs/concepts/containers/images/)
