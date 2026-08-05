# Task 01: Deploy Pods in Kubernetes Cluster

The Nautilus DevOps team is diving into Kubernetes for application management. One team member has a task to create a pod according to the details below:

## Task Requirements

1. Create a pod named `pod-nginx` using the `nginx` image with the latest tag. Ensure to specify the tag as `nginx:latest`.
2. Set the app label to `nginx_app`, and name the container as `nginx-container`.

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

The Pod must have four exact values: the resource name `pod-nginx`, the image `nginx:latest`, the label `app: nginx_app`, and the container name `nginx-container`.

Although `kubectl run` can create a simple Pod, a YAML manifest is clearer for this task because it lets us explicitly define both the Pod label and the container name. The manifest is applied from the jump-host, where `kubectl` is already configured to communicate with the cluster.

### 🔧 Step 1: Create the Pod manifest

```bash
vi pod-nginx.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nginx
  labels:
    app: nginx_app
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
```

> **Why:** `vi` is a terminal text editor. The `i` key enters insert mode so the manifest can be typed or pasted. `Esc` leaves insert mode, and `:wq` writes the file and quits the editor. The YAML uses `apiVersion: v1` for the core Kubernetes API, `kind: Pod` to identify the resource type, and `metadata.name` to assign the Pod name. The `metadata.labels` section assigns the `app` label with the value `nginx_app`. Under `spec.containers`, `name` sets the required container name and `image` specifies the exact image reference `nginx:latest`, including the required `latest` tag.

### 🚀 Step 2: Create the Pod in the cluster

```bash
kubectl apply -f pod-nginx.yaml
```

The lab completed successfully. The expected output on the first application is:

```text
pod/pod-nginx created
```

> **Why:** `kubectl` is the Kubernetes command-line client. `apply` sends the desired configuration to the cluster and creates the resource when it does not exist. The `-f` option tells `kubectl` to read the resource definition from a file, and `pod-nginx.yaml` is the manifest created in the previous step. Applying the manifest creates the Pod with the exact name, label, image, and container name required by the task.

### ✅ Step 3: Verify the Pod and label

The challenge was accepted after the Pod was created, so this verification command was not required during the lab. If a manual check is needed, run:

```bash
kubectl get pod pod-nginx --show-labels
```

The expected result is a row for `pod-nginx` with the label `app=nginx_app` and, after the image finishes downloading, a `Running` status:

```text
NAME        READY   STATUS    RESTARTS   AGE   LABELS
pod-nginx   1/1     Running   0          ...   app=nginx_app
```

> **Why:** `get` retrieves information about a Kubernetes resource. `pod` specifies the resource type, and `pod-nginx` selects the Pod by name. The `--show-labels` option adds the Pod's labels to the output, making it possible to confirm the required `app=nginx_app` label. The `1/1` readiness value indicates that the Pod's one container is ready.

## Best Practices

- **Use a declarative manifest for repeatable configuration.** YAML records the desired Kubernetes state and can be reviewed, reused, and applied again.
- **Specify image tags explicitly.** The challenge requires `nginx:latest`; in production, a fixed version tag is usually safer because `latest` can change over time.
- **Use meaningful labels.** The `app=nginx_app` label identifies the application and can later be used by Services, selectors, and operational commands.
- **Give containers clear names.** `nginx-container` makes the container easy to identify in Pod descriptions, logs, and troubleshooting output.
- **Use a Deployment for long-running production workloads.** This task intentionally creates a standalone Pod, but Deployments are normally preferred because they provide rollout and replacement management.

### 📚 Official Documentation

- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Kubernetes container images](https://kubernetes.io/docs/concepts/containers/images/)
