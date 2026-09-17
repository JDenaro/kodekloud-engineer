# Day 53: Resolve VolumeMounts Issue in Kubernetes

We encountered an issue with our Nginx and PHP-FPM setup on the Kubernetes cluster this morning, which halted its functionality. Investigate and rectify the issue:


The pod name is `nginx-phpfpm` and configmap name is `nginx-config`. Identify and fix the problem.

Once resolved, copy `/home/thor/index.php` file from the `jump host` to the `nginx-container` within the nginx document root. After this, you should be able to access the website using `Website` button on the top bar.

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Specific Requirements:

1. The pod name is `nginx-phpfpm` and configmap name is `nginx-config`. Identify and fix the problem.
2. Once resolved, copy `/home/thor/index.php` file from the `jump host` to the `nginx-container` within the nginx document root.
3. After this, the website must be accessible using the `Website` button on the top bar.

## Solution

The Pod initially reported `2/2 Running`, but this only confirmed that both container processes were healthy. The application was broken because the `shared-files` volume was mounted at `/var/www/html` in PHP-FPM and at `/usr/share/nginx/html` in Nginx, while the `nginx-config` ConfigMap defined `/var/www/html` as the document root. The fix was to mount the shared volume at `/var/www/html` in both containers and recreate the Pod.

### 🔍 Step 1: Inspect the Pod

```bash
kubectl get pod nginx-phpfpm
kubectl describe pod nginx-phpfpm
```

The Pod was running, but the container mount information revealed the inconsistency:

```text
php-fpm-container:
  Mounts:
    /var/www/html from shared-files (rw)

nginx-container:
  Mounts:
    /usr/share/nginx/html from shared-files (rw)
```

> **Why:** `kubectl get pod` provides a quick health summary, while `kubectl describe pod` presents readable details about containers, volumes, mounts, and events. A `Running` state does not validate that application paths and configuration agree, so the mount details must also be inspected.

### 🔎 Step 2: Inspect the Nginx ConfigMap and Pod Definition

```bash
kubectl get configmap nginx-config -o yaml
kubectl get pod nginx-phpfpm -o yaml
```

The ConfigMap showed that Nginx expected the website under `/var/www/html`:

```nginx
root /var/www/html;
index index.html index.htm index.php;
fastcgi_pass 127.0.0.1:9000;
```

However, the Pod mounted `shared-files` at `/usr/share/nginx/html` in `nginx-container`. PHP-FPM already mounted the same volume at the correct `/var/www/html` path.

> **Why:** `-o yaml` displays the complete resource definition. Comparing the ConfigMap's `root` directive with each container's `mountPath` identified the actual cause. Both containers belong to the same Pod, so they can reach each other through `127.0.0.1`; therefore, the FastCGI address was already correct and did not need modification.

### 📝 Step 3: Create a Corrected Pod Manifest

```bash
vi /tmp/nginx-phpfpm.yaml
```

Use the following manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-phpfpm
  labels:
    app: php-app
spec:
  containers:
    - name: php-fpm-container
      image: php:7.2-fpm-alpine
      volumeMounts:
        - name: shared-files
          mountPath: /var/www/html

    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - name: shared-files
          mountPath: /var/www/html
        - name: nginx-config-volume
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf

  volumes:
    - name: shared-files
      emptyDir: {}

    - name: nginx-config-volume
      configMap:
        name: nginx-config
```

> **Why:** `shared-files` is an `emptyDir` volume shared by both containers for the lifetime of the Pod. Mounting it at `/var/www/html` in both containers aligns the shared files with Nginx's document root and PHP-FPM's script path. The `nginx-config-volume` exposes the ConfigMap, while `subPath: nginx.conf` mounts only its `nginx.conf` key over the expected configuration file instead of replacing the entire `/etc/nginx` directory.

#### Why `kubectl edit pod` Cannot Fix This Issue

Running the following command would open the live Pod definition in an editor:

```bash
kubectl edit pod nginx-phpfpm
```

However, Kubernetes does not allow the `volumeMounts` or `volumes` of an existing Pod to be changed. These fields define how the Pod is constructed and are immutable after the Pod has been created. Attempting to save the corrected `mountPath` would therefore be rejected with an error indicating that Pod updates may not change those fields.

`kubectl edit pod` can still be useful for inspecting a Pod or changing one of the limited mutable fields, such as a container image. It cannot rebuild the container filesystems with different mounts. Because `nginx-phpfpm` is a standalone Pod, it must be deleted and recreated from a corrected manifest. If it were managed by a Deployment, the Deployment's Pod template should be edited instead; the Deployment controller would then replace its Pods automatically.

### ♻️ Step 4: Recreate the Pod

```bash
kubectl replace --force -f /tmp/nginx-phpfpm.yaml
```

The command returned:

```text
pod "nginx-phpfpm" deleted from default namespace
pod/nginx-phpfpm replaced
```

> **Why:** A Pod's volume mounts cannot be changed with `kubectl edit pod`. `kubectl replace` reads the complete replacement definition from the file selected by `-f`. The `--force` option deletes the existing standalone Pod and recreates it with the corrected mounts. This causes a brief interruption and should be used carefully outside a temporary lab.

### ⏳ Step 5: Confirm That Both Containers Are Running

```bash
kubectl get pod nginx-phpfpm
```

The corrected Pod reached the expected state:

```text
NAME           READY   STATUS    RESTARTS   AGE
nginx-phpfpm   2/2     Running   0          3m13s
```

> **Why:** The `READY` value `2/2` confirms that both the Nginx and PHP-FPM containers started successfully after the Pod was recreated.

### 📄 Step 6: Copy the PHP File into the Shared Document Root

```bash
kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container
```

> **Why:** `kubectl cp` copies the local file from the jump host into the Pod. The destination uses the `pod:path` format, and `-c nginx-container` explicitly selects the Nginx container. Because `/var/www/html` is backed by `shared-files`, PHP-FPM can immediately access the same file without a second copy.

### ✅ Step 7: Verify the Shared File and Nginx Configuration

```bash
kubectl exec nginx-phpfpm -c nginx-container -- ls -l /var/www/html/index.php
kubectl exec nginx-phpfpm -c php-fpm-container -- ls -l /var/www/html/index.php
kubectl exec nginx-phpfpm -c nginx-container -- nginx -t
```

Both containers reported the same file:

```text
-rw-r--r-- 1 1000 1000 19 Sep 16 18:48 /var/www/html/index.php
-rw-r--r--    1 1000     1000            19 Sep 16 18:48 /var/www/html/index.php
```

Nginx also validated the mounted configuration:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

> **Why:** `kubectl exec` runs a command in a selected container. The `-c` option identifies the target container, while `--` separates the `kubectl` arguments from the in-container command. Seeing `index.php` from both containers proves that the shared volume is mounted correctly. `nginx -t` verifies the syntax and usability of the ConfigMap-provided Nginx configuration. The final functional check is opening the `Website` button and confirming that PHP information is displayed.

## Best Practices

- **Align application paths with volume mounts.** A shared volume may be mounted at different paths in different containers, but each path must match what that container's application configuration expects.
- **Do not equate Pod health with application health.** `Running` and `Ready` show that processes passed their container checks; they do not prove that Nginx can locate and execute the expected PHP file.
- **Edit the controller instead of its Pods.** Pods created by a Deployment should not be edited directly. Update the Deployment's Pod template so that the controller performs the replacement and preserves the declared state.
- **Use a controller for production workloads.** A Deployment normally manages Pod replacement and recovery. Force-replacing a standalone Pod causes downtime and discards data stored in its `emptyDir` volume.
- **Treat `emptyDir` data as temporary.** Its contents survive container restarts but are deleted when the Pod is removed, which is why `index.php` had to be copied after recreating the Pod.

### 📚 Official Documentation

- [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Communicate Between Containers in the Same Pod Using a Shared Volume](https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)
- [kubectl replace](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#replace)
- [kubectl cp](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_cp/)
