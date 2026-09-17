# Day 54: Kubernetes Shared Volumes

We are working on an application that will be deployed on multiple containers within a pod on Kubernetes cluster. There is a requirement to share a volume among the containers to save some temporary data. The Nautilus DevOps team is developing a similar template to replicate the scenario. Below you can find more details about it.

## Specific Requirements:

1. Create a pod named `volume-share-devops`.
2. For the first container, use image `debian` with `latest` tag only and remember to mention the tag i.e `debian:latest`, container should be named as `volume-container-devops-1`, and run a `sleep` command for it so that it remains in running state. Volume `volume-share` should be mounted at path `/tmp/media`.
3. For the second container, use image `debian` with the `latest` tag only and remember to mention the tag i.e `debian:latest`, container should be named as `volume-container-devops-2`, and again run a `sleep` command for it so that it remains in running state. Volume `volume-share` should be mounted at path `/tmp/apps`.
4. Volume name should be `volume-share` of type `emptyDir`.
5. After creating the pod, exec into the first container i.e `volume-container-devops-1`, and just for testing create a file `media.txt` with the content `Welcome to xFusionCorp Industries` under the mounted path of first container i.e `/tmp/media`.
6. The file `media.txt` should be present under the mounted path `/tmp/apps` on the second container `volume-container-devops-2` as well, since they are using a shared volume.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

An `emptyDir` volume can be mounted by multiple containers in the same Pod. Each container may use a different `mountPath`, but the paths expose the same underlying storage. In this lab, `/tmp/media` in the first container and `/tmp/apps` in the second container both refer to the `volume-share` volume.

### 📝 Step 1: Create the Pod Manifest

```bash
vi volume-share-devops.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-share-devops
spec:
  containers:
    - name: volume-container-devops-1
      image: debian:latest
      command:
        - sleep
        - "3600"
      volumeMounts:
        - name: volume-share
          mountPath: /tmp/media

    - name: volume-container-devops-2
      image: debian:latest
      command:
        - sleep
        - "3600"
      volumeMounts:
        - name: volume-share
          mountPath: /tmp/apps

  volumes:
    - name: volume-share
      emptyDir: {}
```

> **Why:** The manifest declares one Pod with two explicitly named containers using `debian:latest`. The `sleep 3600` command provides a long-running foreground process so that Debian does not immediately exit. Each `volumeMounts` entry references `volume-share`, but exposes it at the path required by that container. The Pod-level `volumes` section creates `volume-share` as an `emptyDir`, which provides temporary storage for the lifetime of the Pod.

### 🚀 Step 2: Create the Pod

```bash
kubectl apply -f volume-share-devops.yaml
```

The command returned:

```text
pod/volume-share-devops created
```

> **Why:** `kubectl apply` sends the manifest to the Kubernetes API and creates the declared Pod. The `-f` option identifies the YAML file containing the resource definition.

### 🔍 Step 3: Confirm That Both Containers Are Running

```bash
kubectl get pods
```

The Pod reached the required state:

```text
NAME                  READY   STATUS    RESTARTS   AGE
volume-share-devops   2/2     Running   0          14s
```

> **Why:** `kubectl get pods` displays the current state of the Pods. `READY` showing `2/2` confirms that both Debian containers are running, while `STATUS` showing `Running` confirms that the Pod is operational.

### 📄 Step 4: Create the Test File in the First Container

```bash
kubectl exec -it volume-share-devops -c volume-container-devops-1 -- /bin/bash
```

Inside the container, create and read the file, and then leave the interactive shell:

```bash
echo "Welcome to xFusionCorp Industries" > /tmp/media/media.txt
cat /tmp/media/media.txt
exit
```

The file contained:

```text
Welcome to xFusionCorp Industries
```

> **Why:** `kubectl exec` runs a command inside a container. The `-it` options allocate an interactive terminal, `-c volume-container-devops-1` selects the first container, and `--` separates the `kubectl` options from `/bin/bash`, the command executed inside Debian. The shell redirection operator writes the required text into `/tmp/media/media.txt`, which is stored in the mounted `emptyDir` volume. `cat` confirms the content, and `exit` returns to the jump host.

### ✅ Step 5: Verify the Shared File from the Second Container

```bash
kubectl exec volume-share-devops -c volume-container-devops-2 -- cat /tmp/apps/media.txt
```

The second container returned:

```text
Welcome to xFusionCorp Industries
```

> **Why:** This `kubectl exec` command selects `volume-container-devops-2` and reads `/tmp/apps/media.txt`. The file was originally written through `/tmp/media` in the first container, so retrieving the same content through `/tmp/apps` proves that both mount paths expose the same `volume-share` storage.

## Best Practices

- **Use shared volumes for data, not container layers.** Containers have separate root filesystems, so files that must be available to more than one container should be written to a shared volume.
- **Treat `emptyDir` data as temporary.** The volume survives individual container restarts but is deleted permanently when the Pod is removed from its node.
- **Use descriptive volume and container names.** Clear names make multi-container Pod manifests and troubleshooting output easier to understand.
- **Choose persistent storage for durable data.** Use a PersistentVolume rather than `emptyDir` when data must survive Pod replacement or deletion.

### 📚 Official Documentation

- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Communicate Between Containers in the Same Pod Using a Shared Volume](https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)
- [kubectl exec](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/)
