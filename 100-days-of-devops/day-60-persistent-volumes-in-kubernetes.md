# Day 60: Persistent Volumes in Kubernetes

The Nautilus DevOps team is working on a Kubernetes template to deploy a web application on the cluster. There are some requirements to create/use persistent volumes to store the application code, and the template needs to be designed accordingly. Please find more details below:

## Specific Requirements:

1. Create a `PersistentVolume` named as `pv-nautilus`. Configure the `spec` as storage class should be `manual`, set capacity to `5Gi`, set access mode to `ReadWriteOnce`, volume type should be `hostPath` and set path to `/mnt/devops` (this directory is already created, you might not be able to access it directly, so you need not to worry about it).
2. Create a `PersistentVolumeClaim` named as `pvc-nautilus`. Configure the `spec` as storage class should be `manual`, request `2Gi` of the storage, set access mode to `ReadWriteOnce`.
3. Create a `pod` named as `pod-nautilus`, mount the persistent volume you created with claim name `pvc-nautilus` at document root of the web server, the container within the pod should be named as `container-nautilus` using image `nginx` with `latest` tag only (remember to mention the tag i.e `nginx:latest`).
4. Create a node port type service named `web-nautilus` using node port `30008` to expose the web server running within the pod.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

Kubernetes separates how storage is provided from how an application requests and consumes it. This challenge statically provisions a `hostPath` PersistentVolume, claims part of its capacity with a PersistentVolumeClaim, and mounts the claim at Nginx's document root.

### Understanding PersistentVolumes and PersistentVolumeClaims

A **PersistentVolume (PV)** represents storage that the cluster can offer. It describes the storage implementation and capabilities: where the data resides, total capacity, storage class, and supported access modes. A PV is a cluster-scoped resource, has a lifecycle independent of an individual Pod, and can therefore remain available when a Pod is replaced.

A **PersistentVolumeClaim (PVC)** is an application's request for storage. It describes what the workload needs rather than where the storage physically resides: requested capacity, storage class, and access mode. A PVC belongs to a namespace. A Pod in that namespace consumes the claim without needing to know that this lab uses `/mnt/devops` on the node.

| Concept | PersistentVolume | PersistentVolumeClaim |
| --- | --- | --- |
| Simple analogy | An available storage unit | A reservation for a storage unit |
| Created for | Supplying storage to the cluster | Requesting storage for a workload |
| Scope | Cluster-wide | Namespaced |
| Describes | Real storage, capacity, class, and access modes | Requested size, class, and access modes |
| Used by the Pod | Indirectly | Directly through `claimName` |
| Lab resource | `pv-nautilus` | `pvc-nautilus` |

The relationship in this challenge is:

```text
Node directory /mnt/devops
          ↓ hostPath
PersistentVolume pv-nautilus (5Gi available)
          ↓ one-to-one binding
PersistentVolumeClaim pvc-nautilus (requests 2Gi)
          ↓ claimName
Pod volume nautilus-storage
          ↓ volumeMount
/usr/share/nginx/html inside container-nautilus
```

The PVC can bind to the 5 GiB PV because the volume satisfies the requested class, access mode, and minimum capacity of 2 GiB. Kubernetes assigns the complete PV to the claim; it does not split this PV and give only 2 GiB to the claim. This is why the bound PVC later reports a capacity of `5Gi`.

`ReadWriteOnce`, abbreviated as `RWO`, means the volume can be mounted read-write by one node. It does not necessarily mean only one Pod; multiple Pods on the same node can potentially use an RWO volume when the storage implementation permits it.

### 📝 Step 1: Create the Kubernetes Manifest

```bash
vi nautilus-persistent-volume.yaml
```

Add the following configuration:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nautilus
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/devops
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nautilus
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-nautilus
  labels:
    app: nginx-nautilus
spec:
  containers:
    - name: container-nautilus
      image: nginx:latest
      ports:
        - containerPort: 80
      volumeMounts:
        - name: nautilus-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: nautilus-storage
      persistentVolumeClaim:
        claimName: pvc-nautilus
---
apiVersion: v1
kind: Service
metadata:
  name: web-nautilus
spec:
  type: NodePort
  selector:
    app: nginx-nautilus
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30008
```

> **Why:** The PV uses static provisioning to expose 5 GiB from the node path `/mnt/devops` with the class `manual` and the `ReadWriteOnce` access mode. The PVC requests at least 2 GiB with matching class and access mode values, allowing the Kubernetes control plane to bind the two resources. The Pod's `volumes` entry refers to `pvc-nautilus`, while `volumeMounts` presents that claimed storage at Nginx's document root, `/usr/share/nginx/html`. The Pod label and Service selector both use `app: nginx-nautilus`, enabling the Service to find the Pod. The Service forwards TCP traffic from node port `30008` to port `80` in the container. The `---` markers separate the four resources in one file.

Do not add `type: Directory` under `hostPath` for this lab. The `type` field is optional. When omitted, Kubernetes performs no path-type check before mounting. Setting it to `Directory` would require `/mnt/devops` to pass an explicit existing-directory check on the node; that check failed in this environment and kept the Pod in `ContainerCreating`.

### 🚀 Step 2: Create the Resources

```bash
kubectl apply -f nautilus-persistent-volume.yaml
```

Kubernetes created all four resources:

```text
persistentvolume/pv-nautilus created
persistentvolumeclaim/pvc-nautilus created
pod/pod-nautilus created
service/web-nautilus created
```

> **Why:** `kubectl apply` submits the declarative resources to the Kubernetes API. The `-f` option selects `nautilus-persistent-volume.yaml`. Kubernetes then reconciles the PV, PVC, Pod, and Service described in the file.

### 🔗 Step 3: Verify the PV and PVC Binding

```bash
kubectl get pv pv-nautilus
kubectl get pvc pvc-nautilus
```

The volume and claim bound successfully:

```text
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS
pv-nautilus   5Gi        RWO            Retain           Bound    default/pvc-nautilus   manual

NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS
pvc-nautilus   Bound    pv-nautilus   5Gi        RWO            manual
```

> **Why:** `kubectl get pv` shows the cluster-scoped volume. `STATUS Bound` and `CLAIM default/pvc-nautilus` prove that Kubernetes reserved it for this claim. `kubectl get pvc` shows the namespaced request. Its `VOLUME pv-nautilus` field confirms the reverse side of the one-to-one binding. The claim reports `5Gi` because it receives the whole matching PV, even though it requested a minimum of `2Gi`. `Retain` is the default reclaim policy shown for this statically created PV, meaning its storage is not automatically erased when the claim is removed.

In this lab, Kubernetes briefly reported that a StorageClass object named `manual` did not exist. That warning relates to dynamic provisioning. Static binding still succeeded because an existing PV had the same `storageClassName`, sufficient capacity, and the requested access mode.

### 🌐 Step 4: Verify the Pod and Service

```bash
kubectl get pod pod-nautilus
kubectl get service web-nautilus
```

The web server and NodePort Service became available:

```text
NAME           READY   STATUS    RESTARTS   AGE
pod-nautilus   1/1     Running   0          17s

NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
web-nautilus   NodePort   10.43.48.78   <none>        80:30008/TCP   3m55s
```

> **Why:** `kubectl get pod` confirms that Nginx started and that its only container is ready. `kubectl get service` confirms that `web-nautilus` is a NodePort Service exposing Service port `80` through port `30008` on the node.

### 🔎 Step 5: Confirm the Claim Mount

```bash
kubectl describe pod pod-nautilus
```

The running Pod showed the expected mount and claim:

```text
Mounts:
  /usr/share/nginx/html from nautilus-storage (rw)

Volumes:
  nautilus-storage:
    Type:       PersistentVolumeClaim
    ClaimName:  pvc-nautilus
    ReadOnly:   false
```

> **Why:** `kubectl describe pod` displays container state, mounts, volumes, conditions, and events. The mount proves that the Nginx document root uses `nautilus-storage`, while `ClaimName: pvc-nautilus` proves that the Pod consumes the PVC rather than referring directly to the PV or host path. This abstraction allows the workload manifest to request storage without embedding the underlying storage implementation.

### ✅ Step 6: Test the NodePort

```bash
curl http://localhost:30008
```

Nginx responded through the required NodePort:

```html
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.31.6</center>
</body>
</html>
```

> **Why:** `curl` sends an HTTP request to port `30008` on the node. The response proves that traffic reached the NodePort Service and the Nginx container. The `403 Forbidden` response is expected when the mounted `/mnt/devops` storage does not contain an index file that Nginx can serve; it does not indicate a failure in the Pod, volume binding, mount, or Service routing required by this challenge.

## Best Practices

- **Let applications consume claims, not storage implementations.** Pods should normally reference PVCs so workload definitions remain separate from details such as host paths, NFS servers, or cloud disks.
- **Match storage requirements deliberately.** A static PV must satisfy the PVC's storage class, access mode, volume mode, and minimum requested capacity before binding can occur.
- **Understand that binding is exclusive.** A PV-to-PVC binding is one-to-one. A 5 GiB PV bound to a 2 GiB request is reserved as a whole and is not automatically divided among other claims.
- **Interpret `ReadWriteOnce` correctly.** RWO limits read-write mounting to one node, not necessarily to one Pod. Use `ReadWriteOncePod` with compatible CSI storage when strict single-Pod access is required.
- **Use `hostPath` only for suitable test or node-level workloads.** The data is tied to one node and carries security and availability risks. Production applications normally use CSI-backed, network, or cloud storage.
- **Avoid adding unrequested validation constraints.** `hostPath.type: Directory` requires the path to exist as a directory. Omitting the optional type was necessary in this lab because the environment did not pass that check.
- **Treat HTTP status separately from connectivity.** The Nginx `403` response proved network reachability; it reflected missing web content in the mounted directory rather than a Kubernetes networking failure.

### 📚 Official Documentation

- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Volumes and hostPath Types](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Services and NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport)
