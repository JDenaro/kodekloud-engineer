# Task 07: Deploy ReplicaSet in Kubernetes Cluster

The Nautilus DevOps team is gearing up to deploy applications on a Kubernetes cluster for migration purposes. A team member has been tasked with creating a ReplicaSet outlined below:

## Task Requirements

1. Create a ReplicaSet using `httpd` image with latest tag (ensure to specify as `httpd:latest`) and name it `httpd-replicaset`.
2. Apply labels: `app` as `httpd_app`, `type` as `front-end`.
3. Name the container `httpd-container`. Ensure the replica count is `4`.

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

A ReplicaSet maintains a desired number of identical Pods. This task requires four Pods using the `httpd:latest` image, with the container name `httpd-container` and the labels `app: httpd_app` and `type: front-end`.

The labels must be present both in the ReplicaSet selector and in the Pod template. The selector tells the ReplicaSet which Pods it manages, while the template provides the labels that newly created Pods receive.

### 📝 Step 1: Create the ReplicaSet manifest

```bash
vi httpd-replicaset.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: httpd-replicaset
  labels:
    app: httpd_app
    type: front-end
spec:
  replicas: 4
  selector:
    matchLabels:
      app: httpd_app
      type: front-end
  template:
    metadata:
      labels:
        app: httpd_app
        type: front-end
    spec:
      containers:
        - name: httpd-container
          image: httpd:latest
```

> **Why:** `vi` is a terminal text editor. The `i` key enters insert mode, `Esc` leaves insert mode, and `:wq` saves the file and exits. `apiVersion: apps/v1` selects the stable API version for ReplicaSets, and `kind: ReplicaSet` identifies the resource type. `metadata.name` gives the ReplicaSet the required name. `metadata.labels` labels the ReplicaSet object itself. `spec.replicas: 4` sets the desired number of Pods. `spec.selector.matchLabels` defines the labels used to identify the managed Pods. The `spec.template` section describes the Pods that the ReplicaSet creates, including their labels, container name, and `httpd:latest` image.

The values in `spec.selector.matchLabels` must match the values in `spec.template.metadata.labels`. If they do not match, Kubernetes rejects the ReplicaSet because it would not know which Pods it is responsible for managing.

### 🚀 Step 2: Create the ReplicaSet

```bash
kubectl apply -f httpd-replicaset.yaml
```

The ReplicaSet was created successfully:

```text
thor@jump-host ~$ vi httpd-replicaset.yaml
thor@jump-host ~$ kubectl apply -f httpd-replicaset.yaml
replicaset.apps/httpd-replicaset created
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client, and `apply` sends the desired resource configuration to the cluster. The `-f` option tells `kubectl` to read the configuration from a file, and `httpd-replicaset.yaml` is the manifest created in the previous step. The output confirms that the ReplicaSet resource was accepted and created.

### ✅ Step 3: Verify the four Pods

```bash
kubectl get pods
```

The verification showed four Pods, each ready and running:

```text
thor@jump-host ~$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
httpd-replicaset-bsdrm   1/1     Running   0          21s
httpd-replicaset-c2cqr   1/1     Running   0          21s
httpd-replicaset-q6xln   1/1     Running   0          21s
httpd-replicaset-qv8nk   1/1     Running   0          21s
thor@jump-host ~$
```

> **Why:** `get` retrieves information about Kubernetes resources, and `pods` selects all Pods in the current namespace. The four generated names show that the ReplicaSet created four separate Pods. `1/1` means the one container in each Pod is ready, and `Running` confirms that all four Pods are operational. The ReplicaSet adds the random suffixes to keep the generated Pod names unique.

## Best Practices

- **Keep the selector and template labels identical.** The ReplicaSet uses the selector to identify the Pods that it owns and maintains.
- **Use a ReplicaSet to maintain a fixed number of Pods.** If one of the four Pods stops running, the ReplicaSet controller creates a replacement to restore the desired count.
- **Use explicit image tags.** The task requires `httpd:latest`; production workloads should usually use a fixed version tag for predictable deployments.
- **Verify both count and readiness.** Four Pod names alone are not enough; each Pod should show `1/1` ready and `Running` status.
- **Prefer Deployments for most application workloads.** A Deployment manages ReplicaSets and provides rolling updates and rollback support, while a directly managed ReplicaSet is appropriate for tasks that specifically require one.
- **Avoid overlapping selectors.** Two controllers with the same selector can attempt to manage the same Pods and produce unpredictable behavior.

### 📚 Official Documentation

- [Kubernetes ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
