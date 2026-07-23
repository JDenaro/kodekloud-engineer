# Task 03: Setup Kubernetes Namespaces and PODs

The Nautilus DevOps team is planning to deploy some micro services on Kubernetes platform. The team has already set up a Kubernetes cluster and now they want to set up some namespaces, deployments etc. Based on the current requirements, the team has shared some details as below:

## Task Requirements

1. Create a namespace named `dev` and deploy a POD within it. Name the pod `dev-nginx-pod` and use the nginx image with the latest tag. Ensure to specify the tag as `nginx:latest`.

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

This task can be completed with either imperative commands or declarative YAML manifests. The imperative method is quick for an interactive lab. The YAML method records the desired namespace and Pod configuration in files that can be reviewed and reused.

The namespace must be created before the Pod, and the Pod must explicitly specify `dev` as its namespace. Creating a namespace does not automatically change the namespace used by later `kubectl` commands.

## Imperative Method

### 🚀 Step 1: Create the `dev` namespace

```bash
kubectl create namespace dev
```

The command succeeded:

```text
thor@jump-host ~$ kubectl create namespace dev
namespace/dev created
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client. `create namespace` creates a namespace resource, and `dev` is the required namespace name. A namespace provides a logical boundary for Kubernetes resources, helping teams separate applications and environments within the same cluster.

### 🚀 Step 2: Create the Pod in the `dev` namespace

```bash
kubectl run dev-nginx-pod --image=nginx:latest --namespace=dev
```

The command succeeded:

```text
thor@jump-host ~$ kubectl run dev-nginx-pod --image=nginx:latest --namespace=dev
pod/dev-nginx-pod created
thor@jump-host ~$
```

> **Why:** `run` creates a Pod from a container image. `dev-nginx-pod` is the required Pod name, and `--image=nginx:latest` specifies the `nginx` image with the required `latest` tag. The `--namespace=dev` option selects the `dev` namespace for this request; without it, the Pod would be created in the current default namespace instead.

## Declarative Method

### 📝 Step 1: Create the namespace manifest

```bash
vi dev-namespace.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

> **Why:** `vi` is a terminal text editor. The `i` key enters insert mode, `Esc` leaves insert mode, and `:wq` saves the file and exits. `apiVersion: v1` selects the core Kubernetes API, `kind: Namespace` identifies the resource type, and `metadata.name` assigns the namespace name `dev`.

### 📝 Step 2: Create the Pod manifest

```bash
vi dev-nginx-pod.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dev-nginx-pod
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

> **Why:** `kind: Pod` identifies the workload resource, and `metadata.name` sets the required Pod name. The `metadata.namespace` field places the Pod in `dev`; this field is necessary because YAML manifests do not automatically inherit the namespace created in a previous file. Under `spec.containers`, the `image` field selects `nginx:latest`, including the required tag.

### 🚀 Step 3: Apply the namespace and Pod manifests

```bash
kubectl apply -f dev-namespace.yaml
kubectl apply -f dev-nginx-pod.yaml
```

Expected output on a new cluster:

```text
namespace/dev created
pod/dev-nginx-pod created
```

> **Why:** `apply` sends the desired state in a manifest to the Kubernetes API. The `-f` option tells `kubectl` to read a resource definition from a file. Applying the namespace first guarantees that the target namespace exists before Kubernetes tries to create the Pod inside it. Applying the Pod manifest then creates `dev-nginx-pod` in `dev`.

### ✅ Step 4: Verify the Pod in the correct namespace

The lab was completed successfully with the imperative commands, so this additional check was not required during the session. If a manual verification is needed, run:

```bash
kubectl get pod dev-nginx-pod --namespace=dev
```

The expected result includes the Pod name and the `dev` namespace-scoped Pod in a `Running` state after the image has been downloaded:

```text
NAME            READY   STATUS    RESTARTS   AGE
dev-nginx-pod   1/1     Running   0          ...
```

> **Why:** `get` retrieves information about a Kubernetes resource. `pod` selects the Pod resource type, and `dev-nginx-pod` selects the resource by name. The `--namespace=dev` option ensures the command looks in the namespace where the Pod was created rather than in `default`.

## Best Practices

- **Use namespaces to separate environments.** A namespace such as `dev` provides a clear boundary for development workloads inside a shared cluster.
- **Always specify the target namespace.** Use `--namespace=dev` for imperative commands and `metadata.namespace: dev` in YAML so resources are not accidentally created in `default`.
- **Apply namespace manifests before workload manifests.** The target namespace must exist before a namespaced Pod can be created.
- **Use YAML for repeatable deployments.** Declarative files make the desired state visible, reviewable, and reusable across environments.
- **Specify image tags explicitly.** The task requires `nginx:latest`; production workloads should generally use a fixed version tag for predictable updates.
- **Use a controller for production workloads.** This task intentionally creates a standalone Pod, but a Deployment is normally preferred for application workloads that need replacement and rollout management.

### 📚 Official Documentation

- [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [`kubectl create namespace` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_namespace/)
- [`kubectl run` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_run/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
