# Day 62: Manage Secrets in Kubernetes

The Nautilus DevOps team is working to deploy some tools in Kubernetes cluster. Some of the tools are licence based so that licence information needs to be stored securely within Kubernetes cluster. Therefore, the team wants to utilize Kubernetes secrets to store those secrets. Below you can find more details about the requirements:

## Specific Requirements:

1. We already have a secret key file `media.txt` under the `/opt/` directory. Create a `generic secret` named `media`, it should contain the password/license-number present in `media.txt` file.
2. Also create a `pod` named `secret-nautilus`.
3. Configure pod's `spec` as container name should be `secret-container-nautilus`, image should be `ubuntu` with `latest` tag (remember to mention the tag with image). Use `sleep` command for container so that it remains in running state. Consume the created secret and mount it under `/opt/apps` within the container.
4. To verify you can exec into the container `secret-container-nautilus`, to check the secret key under the mounted path `/opt/apps`. Before hitting the `Check` button please make sure pod/pods are in running state, also validation can take some time to complete so keep patience.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

A Kubernetes Secret stores a small amount of confidential information separately from a Pod manifest or container image. In this challenge, `kubectl` reads the existing licence file into an `Opaque` Secret. Kubernetes then projects the Secret key into the container as a read-only file.

The data flow is:

```text
/opt/media.txt on the jump host
             ↓ --from-file
Secret media with key media.txt
             ↓ Secret volume
/opt/apps/media.txt inside the container
```

### 🔐 Step 1: Create the Generic Secret

```bash
kubectl create secret generic media --from-file=/opt/media.txt
```

Kubernetes created the Secret:

```text
secret/media created
```

> **Why:** `kubectl create secret generic` creates an `Opaque` Secret for arbitrary user-defined data. `media` is the required resource name. The `--from-file` option reads `/opt/media.txt` and stores its contents as the value. Because no explicit key name appears before the path, `kubectl` uses the file's basename, `media.txt`, as the Secret key. This avoids copying the confidential value into shell history or a YAML manifest.

The equivalent explicit form would be `--from-file=media.txt=/opt/media.txt`. That longer form is useful when the key stored in Kubernetes should have a different name from the source file.

### 🔍 Step 2: Inspect the Secret Safely

```bash
kubectl describe secret media
```

The description confirmed the Secret type, key, and value size without printing the licence:

```text
Name:         media
Namespace:    default
Type:         Opaque

Data
====
media.txt:  7 bytes
```

> **Why:** `kubectl describe secret` displays metadata and the names and sizes of stored keys, but not their values. `Opaque` is the generic Secret type. The `media.txt` entry confirms that `--from-file` used the source filename as the key and stored its seven-byte content.

### 📝 Step 3: Create the Pod Manifest

```bash
vi secret-nautilus.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-nautilus
spec:
  containers:
    - name: secret-container-nautilus
      image: ubuntu:latest
      command:
        - /bin/bash
        - -c
        - sleep 3600
      volumeMounts:
        - name: media-secret-volume
          mountPath: /opt/apps
          readOnly: true
  volumes:
    - name: media-secret-volume
      secret:
        secretName: media
```

> **Why:** The manifest creates the required Pod and container with the explicit `ubuntu:latest` image. Bash receives `sleep 3600` through `-c`, keeping the container active for one hour so it can be inspected. The Pod-level `volumes` entry sources `media-secret-volume` from the Secret named `media`. The matching `volumeMounts` entry presents that volume at `/opt/apps`; `readOnly: true` communicates that the application should not modify confidential input. Kubernetes turns each Secret key into a file, so `media.txt` becomes `/opt/apps/media.txt`.

### 🚀 Step 4: Create the Pod

```bash
kubectl apply -f secret-nautilus.yaml
```

> **Why:** `kubectl apply` submits the declarative Pod configuration to the Kubernetes API. The `-f` option identifies `secret-nautilus.yaml` as the manifest to apply. Because the referenced Secret already exists, the kubelet can mount it while preparing the container.

### ✅ Step 5: Verify the Running Pod and Secret Mount

```bash
kubectl get pods
kubectl describe pod secret-nautilus
```

The Pod reached the required running state:

```text
NAME              READY   STATUS    RESTARTS   AGE
secret-nautilus   1/1     Running   0          25s
```

Its description confirmed the container, command, mount, and Secret source:

```text
Containers:
  secret-container-nautilus:
    Image:         ubuntu:latest
    Command:
      /bin/bash
      -c
      sleep 3600
    State:          Running
    Ready:          True
    Mounts:
      /opt/apps from media-secret-volume (ro)

Volumes:
  media-secret-volume:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  media
```

> **Why:** `kubectl get pods` provides a concise status check; `1/1 Running` proves that the only container is ready. `kubectl describe pod` shows the live container configuration, mounted volumes, conditions, and events. `(ro)` confirms the read-only mount, while `SecretName: media` proves that the volume uses the required Secret.

### 🔎 Step 6: Verify the Projected Secret File

```bash
kubectl exec secret-nautilus -c secret-container-nautilus -- ls -l /opt/apps
```

Kubernetes projected the key as a file:

```text
total 0
lrwxrwxrwx 1 root root 16 Sep 18 12:41 media.txt -> ..data/media.txt
```

> **Why:** `kubectl exec` runs a command in the existing Pod. The `-c` option selects `secret-container-nautilus`, and `--` separates `kubectl` arguments from the `ls -l /opt/apps` command executed in the container. The `media.txt` symbolic link is normal for Kubernetes projected volumes: it points to the current version stored under `..data`, allowing Kubernetes to update mounted Secret data atomically.

The value was verified locally with:

```bash
kubectl exec secret-nautilus -c secret-container-nautilus -- cat /opt/apps/media.txt
```

The output matched the original `/opt/media.txt` value. It is intentionally omitted from this guide so the licence is not stored in Git history.

> **Why:** `cat` reads the projected file from the required mount path, proving that Kubernetes stored the source file content in the Secret and made it available to the container. Secret values should not be copied into documentation, terminal recordings, screenshots, logs, or source control.

## Best Practices

- **Do not confuse Base64 with encryption.** Kubernetes represents values in a Secret's `data` field using Base64, which is reversible encoding and does not provide confidentiality by itself.
- **Enable encryption at rest.** Kubernetes Secrets are stored unencrypted in etcd by default unless the cluster administrator configures API data encryption.
- **Apply least-privilege RBAC.** Restrict `get`, `list`, and `watch` permissions for Secrets to only the users and service accounts that require them.
- **Keep values out of manifests and shell history.** Creating a Secret from an existing protected file avoids embedding credentials directly in YAML or command arguments.
- **Mount only where required.** Only containers that need the licence should receive the Secret volume. A Pod must reference a Secret explicitly before its containers can access it.
- **Prefer read-only file mounts.** Secret volumes are inherently read-only, and declaring `readOnly: true` makes the intended usage clear to readers.
- **Avoid exposing values during verification.** Use `kubectl describe secret` and directory listings for routine checks. Read the value only when necessary and never record it in Git.
- **Plan for secret rotation.** Kubernetes eventually updates mounted Secret volumes when their source changes; applications should be able to reread files when rotation is required.

### 📚 Official Documentation

- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Managing Secrets Using kubectl](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [kubectl create secret generic](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_secret_generic/)
- [Secret Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#secret)
- [Good Practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
