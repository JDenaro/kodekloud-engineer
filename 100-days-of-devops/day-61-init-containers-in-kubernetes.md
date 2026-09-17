# Day 61: Init Containers in Kubernetes

There are some applications that need to be deployed on Kubernetes cluster and these apps have some pre-requisites where some configurations need to be changed before deploying the app container. Some of these changes cannot be made inside the images so the DevOps team has come up with a solution to use init containers to perform these tasks during deployment. Below is a sample scenario that the team is going to test first.

## Specific Requirements:

1. Create a `Deployment` named as `ic-deploy-devops`.
2. Configure `spec` as replicas should be `1`, labels `app` should be `ic-devops`, template's metadata lables `app` should be the same `ic-devops`.
3. The `initContainers` should be named as `ic-msg-devops`, use image `fedora` with `latest` tag and use command `'/bin/bash'`, `'-c'` and `'echo Init Done - Welcome to xFusionCorp Industries > /ic/news'`. The volume mount should be named as `ic-volume-devops` and mount path should be `/ic`.
4. Main container should be named as `ic-main-devops`, use image `fedora` with `latest` tag and use command `'/bin/bash'`, `'-c'` and `'while true; do cat /ic/news; sleep 5; done'`. The volume mount should be named as `ic-volume-devops` and mount path should be `/ic`.
5. Volume to be named as `ic-volume-devops` and it should be an emptyDir type.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

An init container performs setup work before a Pod's application containers start. It must run to completion successfully; otherwise, Kubernetes does not start the main container. In this challenge, the init container writes a message to a shared `emptyDir` volume, and the main container reads that prepared file every five seconds.

The startup sequence is:

```text
Pod starts
    ↓
ic-msg-devops writes /ic/news
    ↓
init container exits successfully
    ↓
ic-main-devops starts
    ↓
main container reads /ic/news every five seconds
```

The two containers do not run at the same time. They exchange the file through `ic-volume-devops`, which Kubernetes mounts at `/ic` in both containers.

### 📝 Step 1: Create the Deployment Manifest

```bash
vi ic-deploy-devops.yaml
```

Add the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ic-deploy-devops
  labels:
    app: ic-devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ic-devops
  template:
    metadata:
      labels:
        app: ic-devops
    spec:
      initContainers:
        - name: ic-msg-devops
          image: fedora:latest
          command:
            - /bin/bash
            - -c
            - echo Init Done - Welcome to xFusionCorp Industries > /ic/news
          volumeMounts:
            - name: ic-volume-devops
              mountPath: /ic
      containers:
        - name: ic-main-devops
          image: fedora:latest
          command:
            - /bin/bash
            - -c
            - while true; do cat /ic/news; sleep 5; done
          volumeMounts:
            - name: ic-volume-devops
              mountPath: /ic
      volumes:
        - name: ic-volume-devops
          emptyDir: {}
```

> **Why:** The Deployment maintains the required single replica. The Deployment selector and Pod template label both use `app: ic-devops`, allowing the Deployment to manage its Pod. `initContainers` defines setup containers that must finish before regular entries under `containers` begin. Both containers use `fedora:latest`, start Bash with `/bin/bash`, and pass the command string through `-c`. The init command redirects the required message into `/ic/news`, while the main command repeatedly reads that file and waits five seconds between reads. Both `volumeMounts` refer to `ic-volume-devops` at `/ic`, and the Pod-level `volumes` section creates that shared storage as an `emptyDir`.

An `emptyDir` volume is created for a Pod and initially contains no data. Containers in that Pod can share files through it. Its data survives individual container restarts but is removed when the Pod itself is permanently removed.

### 🚀 Step 2: Create the Deployment

```bash
kubectl apply -f ic-deploy-devops.yaml
```

Kubernetes created the Deployment:

```text
deployment.apps/ic-deploy-devops created
```

> **Why:** `kubectl apply` sends the declarative configuration to the Kubernetes API. The `-f` option identifies `ic-deploy-devops.yaml` as the manifest to apply. The Deployment controller then creates a ReplicaSet and one Pod from its template.

### ⏳ Step 3: Wait for the Deployment

```bash
kubectl rollout status deployment/ic-deploy-devops
```

The rollout completed successfully:

```text
deployment "ic-deploy-devops" successfully rolled out
```

> **Why:** `kubectl rollout status` watches the current Deployment revision until its requested replica becomes available. The Deployment cannot finish successfully until the init container exits with code zero and the main container starts and becomes ready.

### 🔍 Step 4: Check the Deployment and Pod

```bash
kubectl get deployment ic-deploy-devops
kubectl get pods
```

The Deployment and its Pod were healthy:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
ic-deploy-devops   1/1     1            1           42s

NAME                               READY   STATUS    RESTARTS   AGE
ic-deploy-devops-59f7f4b6f-k4h5r   1/1     Running   0          47s
```

> **Why:** `kubectl get deployment` confirms that the desired replica is ready, current, and available. `kubectl get pods` shows the generated Pod in the `Running` state. The `READY 1/1` value counts the application container that remains active; the completed init container is reported separately and does not increase this count.

### 🔎 Step 5: Verify the Container Sequence and Shared Volume

```bash
kubectl describe pod ic-deploy-devops-59f7f4b6f-k4h5r
```

The init container completed successfully before the main container started:

```text
Init Containers:
  ic-msg-devops:
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Finished:     Thu, 17 Sep 2026 17:19:49 +0000

Containers:
  ic-main-devops:
    State:          Running
      Started:      Thu, 17 Sep 2026 17:19:51 +0000
```

Both containers mounted the shared volume, and the Pod reported its temporary lifetime:

```text
Mounts:
  /ic from ic-volume-devops (rw)

Volumes:
  ic-volume-devops:
    Type: EmptyDir (a temporary directory that shares a pod's lifetime)
```

> **Why:** `kubectl describe pod` displays detailed status for init containers, application containers, mounts, volumes, conditions, and events. `Completed` with exit code `0` proves that `ic-msg-devops` finished successfully. Its finish time precedes the main container's start time, confirming the required ordering. The `/ic` mounts and `EmptyDir` volume prove that the file created by the init container is available to the main container.

### ✅ Step 6: Verify the Prepared Message

```bash
kubectl logs deployment/ic-deploy-devops -c ic-main-devops
```

The main container repeatedly read the file prepared by the init container:

```text
Init Done - Welcome to xFusionCorp Industries
Init Done - Welcome to xFusionCorp Industries
Init Done - Welcome to xFusionCorp Industries
```

> **Why:** `kubectl logs` retrieves container standard output. Specifying `deployment/ic-deploy-devops` lets Kubernetes select a Pod managed by the Deployment, while `-c ic-main-devops` selects the main container explicitly. The repeated message proves that the init container created `/ic/news`, that both containers shared the same volume, and that the main container's five-second loop remained operational.

## Best Practices

- **Use init containers for finite prerequisites.** They are suitable for generating configuration, waiting for dependencies, setting permissions, or downloading files before an application starts.
- **Keep initialization separate from the application image.** A dedicated init image can contain setup tools that the runtime image does not need, reducing coupling and the application's attack surface.
- **Make initialization idempotent.** Kubernetes may rerun an init container when a Pod restarts, so its command should remain safe if files or directories already exist.
- **Share initialization results through volumes.** Containers in a Pod do not share their image filesystems, but they can exchange generated files through a commonly mounted volume.
- **Understand `emptyDir` lifetime.** Its contents survive a container restart within the same Pod but disappear when that Pod is deleted or replaced.
- **Do not confuse init containers with sidecars.** A regular init container completes before the application starts; a sidecar continues running alongside the main application.
- **Pin image versions in production.** The challenge requires `fedora:latest`, but immutable or fixed tags provide more predictable production deployments.

### 📚 Official Documentation

- [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Volumes and emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
