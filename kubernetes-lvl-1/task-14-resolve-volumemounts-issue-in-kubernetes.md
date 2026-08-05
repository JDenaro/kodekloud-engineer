# Task 14: Resolve VolumeMounts Issue in Kubernetes

We encountered an issue with our Nginx and PHP-FPM setup on the Kubernetes cluster this morning, which halted its functionality. Investigate and rectify the issue:

The pod name is nginx-phpfpm and configmap name is nginx-config. Identify and fix the problem.

Once resolved, copy /home/thor/index.php file from the jump host to the nginx-container within the nginx document root. After this, you should be able to access the website using Website button on the top bar.

Note: The kubectl utility on the jump-host has been configured to work with the Kubernetes cluster.

## Task Requirements

1. Investigate the existing Pod named `nginx-phpfpm` and the ConfigMap named `nginx-config`.
2. Identify and fix the volume-mount issue affecting the Nginx and PHP-FPM setup.
3. Copy `/home/thor/index.php` from the jump host into the `nginx-container` at the Nginx document root.
4. Confirm that the website is accessible using the Website button.

## Solution

The Pod contained two running containers and one shared `emptyDir` volume. The problem was not visible from the Pod status alone: both containers reported `Running`, but they mounted the shared volume at different paths.

The ConfigMap configured Nginx to use `/var/www/html` as its document root. The `nginx-container` already mounted the shared volume at `/var/www/html`, but `php-fpm-container` mounted the same volume at `/usr/share/nginx/html`. The PHP-FPM mount had to be changed to `/var/www/html` so both containers and the Nginx configuration used the same path.

Because a running Pod does not allow its `volumeMounts` to be changed directly, the Pod definition was exported, edited, and recreated. The application file was copied only after the new Pod was ready because the `emptyDir` volume is recreated with the Pod.

### 🔎 Step 1: Check the Pod status

```bash
kubectl get pod nginx-phpfpm
```

The Pod appeared healthy at the container level:

```text
thor@jump-host ~$ kubectl get pod nginx-phpfpm
NAME           READY   STATUS    RESTARTS   AGE
nginx-phpfpm   2/2     Running   0          29m
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client. `get` retrieves resource information, `pod` selects the Pod resource type, and `nginx-phpfpm` selects the Pod by name. `2/2` and `Running` confirm that both containers started, but they do not prove that the containers are using compatible filesystem paths, so we must inspect the mounts next.

### 🔎 Step 2: Inspect the Pod volume mounts

```bash
kubectl describe pod nginx-phpfpm
```

The relevant part of the output showed the path mismatch:

```text
Containers:
  php-fpm-container:
    Image:          php:7.2-fpm-alpine
    State:          Running
    Ready:          True
    Mounts:
      /usr/share/nginx/html from shared-files (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-t25vt (ro)
  nginx-container:
    Image:          nginx:latest
    State:          Running
    Ready:          True
    Mounts:
      /etc/nginx/nginx.conf from nginx-config-volume (rw,path="nginx.conf")
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-t25vt (ro)
      /var/www/html from shared-files (rw)
Volumes:
  shared-files:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
  nginx-config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      nginx-config
```

> **Why:** `describe` displays detailed information about a resource, including container states, volume mounts, volumes, and events. The same `shared-files` volume was mounted at `/usr/share/nginx/html` in PHP-FPM but at `/var/www/html` in Nginx. A shared volume is useful only when cooperating containers mount it at paths that their application configuration expects.

### ⚙️ Step 3: Inspect the Nginx ConfigMap

```bash
kubectl get configmap nginx-config -o yaml
```

The ConfigMap contained this Nginx configuration:

```yaml
apiVersion: v1
data:
  nginx.conf: |
    events {
    }
    http {
      server {
        listen 8099 default_server;
        listen [::]:8099 default_server;

        # Set nginx to serve files from the shared volume!
        root /var/www/html;
        index  index.html index.htm index.php;
        server_name _;
        location / {
          try_files $uri $uri/ =404;
        }
        location ~ \.php$ {
          include fastcgi_params;
          fastcgi_param REQUEST_METHOD $request_method;
          fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
          fastcgi_pass 127.0.0.1:9000;
        }
      }
    }
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
```

> **Why:** `get configmap` retrieves the ConfigMap, and `-o yaml` prints its complete resource definition in YAML format. The `root /var/www/html` line is the Nginx document root. Since `nginx-container` already used that path, the correct fix was to change the PHP-FPM mount, not the ConfigMap or the Nginx mount.

### 📝 Step 4: Export the Pod definition

```bash
kubectl get pod nginx-phpfpm -o yaml > nginx-phpfpm.yaml
```

The command returned to the prompt without printing output:

```text
thor@jump-host ~$ kubectl get pod nginx-phpfpm -o yaml > nginx-phpfpm.yaml
thor@jump-host ~$
```

> **Why:** `-o yaml` requests the Pod definition in YAML format, and `>` redirects that output into `nginx-phpfpm.yaml`. Exporting the live definition preserves the existing containers, volumes, ConfigMap reference, and metadata while giving us a file to edit.

### 🛠️ Step 5: Correct the PHP-FPM mount path

```bash
vi nginx-phpfpm.yaml
```

Inside the `php-fpm-container` section, change only:

```yaml
- mountPath: /usr/share/nginx/html
  name: shared-files
```

to:

```yaml
- mountPath: /var/www/html
  name: shared-files
```

Do not change the `nginx-container` mount, which already uses `/var/www/html`. Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** Both containers must see the shared `emptyDir` volume at the same application path. After this change, PHP-FPM, Nginx, and the `root` directive in `nginx.conf` all refer to `/var/www/html`.

### ♻️ Step 6: Recreate the Pod with the corrected definition

```bash
kubectl replace --force -f nginx-phpfpm.yaml
```

The Pod was replaced successfully:

```text
thor@jump-host ~$ kubectl replace --force -f nginx-phpfpm.yaml
pod "nginx-phpfpm" deleted from default namespace
pod/nginx-phpfpm replaced
thor@jump-host ~$
```

> **Why:** `replace` updates a resource from a file, and `-f nginx-phpfpm.yaml` selects the edited definition. `--force` deletes and recreates the Pod because `volumeMounts` are not mutable fields of a running Pod. This replacement also creates a new `emptyDir` volume, so files must be copied again after the Pod becomes ready.

### ✅ Step 7: Confirm that the recreated Pod is ready

```bash
kubectl get pods
```

The new Pod returned to the ready state:

```text
thor@jump-host ~$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
nginx-phpfpm   2/2     Running   0          29s
thor@jump-host ~$
```

> **Why:** `get pods` lists Pods in the current namespace. `2/2` confirms that both PHP-FPM and Nginx are ready, while `Running` confirms that the recreated Pod is operational and ready to receive the application file.

### 🌐 Step 8: Copy the PHP file and access the website

```bash
kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container
```

The copy completed without an error message:

```text
thor@jump-host ~$ kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container
thor@jump-host ~$
```

Use the **Website** button in the top bar to access the application.

> **Why:** `kubectl cp` copies files between the local machine and a container. `/home/thor/index.php` is the source file on the jump host, while `nginx-phpfpm:/var/www/html/index.php` identifies the Pod and destination path. The `-c nginx-container` option selects the Nginx container. The destination matches the document root from `nginx.conf`, so Nginx can serve the PHP file and forward PHP requests to PHP-FPM through `127.0.0.1:9000`.

## Root Cause

The `shared-files` volume was mounted at different paths:

```text
php-fpm-container: /usr/share/nginx/html
nginx-container:   /var/www/html
nginx.conf root:   /var/www/html
```

Because the PHP-FPM container used a different path, the Nginx and PHP-FPM processes did not share the application files at the path expected by the configuration. The fix was to recreate the Pod with the PHP-FPM mount changed to `/var/www/html`.

## Best Practices

- **Align application configuration and mount paths.** The Nginx `root`, PHP-FPM working path, and shared volume mounts must describe the same directory.
- **Inspect the Pod instead of trusting its status.** A Pod can show `Running` while the application still has a configuration or filesystem-path problem.
- **Use shared volumes deliberately.** Containers in the same Pod can share an `emptyDir`, but each container must mount it at the path its process expects.
- **Remember Pod immutability.** Changes to fields such as `volumeMounts` require replacing the Pod or updating the workload template that creates it.
- **Copy files after replacing an `emptyDir` Pod.** `emptyDir` data belongs to the Pod lifetime and is lost when the Pod is deleted and recreated.
- **Use `-c` when a Pod has multiple containers.** Selecting `nginx-container` ensures the application file is copied to the intended container.
- **Avoid changing a working ConfigMap unnecessarily.** The ConfigMap already used `/var/www/html`, so correcting the inconsistent PHP-FPM mount was the smaller and safer fix.

### 📚 Official Documentation

- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Pod update and replacement](https://kubernetes.io/docs/concepts/workloads/pods/#pod-update-and-replacement)
- [Kubernetes Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [`kubectl get` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
- [`kubectl describe` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [`kubectl replace` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_replace/)
- [`kubectl cp` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_cp/)
