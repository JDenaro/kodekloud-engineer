# Task 02: Deploy Applications with Kubernetes Deployments

The Nautilus DevOps team is delving into Kubernetes for app management. One team member needs to create a deployment following these details:

## Task Requirements

1. Create a deployment named `nginx` to deploy the application `nginx` using the image `nginx:latest` (ensure to specify the tag)

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

This task can be completed in two valid ways. The imperative method uses one `kubectl` command and is convenient for a quick lab operation. The declarative method stores the desired Deployment configuration in a YAML file and applies it to the cluster, making the configuration easier to review and reuse.

Both methods create a Deployment named `nginx`, so they are alternatives for the same task. They should not be run one after the other on a fresh cluster unless the second command is intentionally being used to manage the resource created by the first one.

## Imperative Method

### 🚀 Step 1: Create the Deployment with `kubectl create`

```bash
kubectl create deployment nginx --image=nginx:latest
```

The expected output when the Deployment does not already exist is:

```text
deployment.apps/nginx created
```

> **Why:** `kubectl` is the Kubernetes command-line client. `create deployment` creates a Deployment resource with the specified name. `nginx` is the required Deployment name, and `--image=nginx:latest` defines the container image, including the required `latest` tag. A Deployment manages the Pod created for the application and uses one replica by default when no replica count is specified.

## Declarative Method

### 📝 Step 1: Create the Deployment manifest

```bash
vi nginx-deployment.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

> **Why:** `vi` is a terminal text editor. The `i` key enters insert mode, `Esc` leaves insert mode, and `:wq` saves the file and exits. `apiVersion: apps/v1` selects the stable API version for Deployments, while `kind: Deployment` identifies the Kubernetes resource type. `metadata.name` gives the Deployment the required name. `replicas: 1` requests one Pod. The selector identifies the Pods managed by the Deployment, and it must match the `app: nginx` label in the Pod template. The template describes the Pod that the Deployment creates, including the `nginx` container and the exact image `nginx:latest`.

### 🚀 Step 2: Apply the Deployment manifest

```bash
kubectl apply -f nginx-deployment.yaml
```

The lab was completed successfully with the declarative method:

```text
thor@jump-host ~$ vi nginx-deployment.yaml
thor@jump-host ~$ kubectl aply -f nginx-deployment.yaml
error: unknown command "aply" for "kubectl"

Did you mean this?
        apply
thor@jump-host ~$ kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx created
thor@jump-host ~$
```

The first attempt contained the typo `aply`. `kubectl` rejected it and suggested the correct subcommand, `apply`. The corrected command created the Deployment successfully.

> **Why:** `apply` sends the desired state from a YAML manifest to the Kubernetes API. The `-f` option means that the resource definition is read from a file, and `nginx-deployment.yaml` is the manifest created in the previous step. If the Deployment does not exist, the command creates it; if it already exists, `apply` updates it to match the manifest. The output `deployment.apps/nginx created` confirms that Kubernetes accepted the resource.

### ✅ Step 3: Verify the Deployment

The lab validator accepted the Deployment after the corrected `kubectl apply` command. If a manual check is needed, run:

```bash
kubectl get deployment nginx
```

The expected result includes the Deployment named `nginx` with one desired, current, and ready replica:

```text
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           ...
```

> **Why:** `get` retrieves information about a Kubernetes resource. `deployment` specifies the resource type, and `nginx` selects the Deployment by name. The `READY`, `UP-TO-DATE`, and `AVAILABLE` columns show whether the Deployment has created and made its requested Pod available.

## Best Practices

- **Use the imperative method for quick experiments.** `kubectl create deployment` is concise and useful when the resource only needs a few straightforward settings.
- **Use the declarative method for repeatable work.** A YAML manifest can be stored in version control, reviewed, and reapplied consistently.
- **Specify image tags explicitly.** The task requires `nginx:latest`; for production workloads, a fixed version tag is generally safer because the `latest` tag can point to different image versions over time.
- **Keep Deployment selectors aligned with Pod labels.** The `spec.selector.matchLabels` values must match the labels in `spec.template.metadata.labels`, or the Deployment cannot correctly manage its Pods.
- **Use one creation method intentionally.** Running both methods with the same name on a new cluster causes the second operation to find an existing Deployment instead of creating an independent resource.
- **Read command suggestions after typos.** The `kubectl` error clearly suggested `apply`, allowing the command to be corrected without changing the cluster.

### 📚 Official Documentation

- [`kubectl create deployment` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_deployment/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
