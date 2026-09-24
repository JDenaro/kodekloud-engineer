# Day 67: Deploy Guest Book App on Kubernetes

The Nautilus Application development team has finished development of one of the applications and it is ready for deployment. It is a guestbook application that will be used to manage entries for guests/visitors. As per discussion with the DevOps team, they have finalized the infrastructure that will be deployed on Kubernetes cluster. Below you can find more details about it.

## Specific Requirements:

`BACK-END TIER`

1. Create a deployment named `redis-master` for Redis master.

   a.) Replicas count should be `1`.

   b.) Container name should be `master-redis-xfusion` and it should use image `redis`.

   c.) Request resources as `CPU` should be `100m` and `Memory` should be `100Mi`.

   d.) Container port should be redis default port i.e `6379`.
2. Create a service named `redis-master` for Redis master. Port and targetPort should be Redis default port i.e `6379`.
3. Create another deployment named `redis-slave` for Redis slave.

   a.) Replicas count should be `2`.

   b.) Container name should be `slave-redis-xfusion` and it should use `gcr.io/google_samples/gb-redisslave:v3` image.

   c.) Requests resources as `CPU` should be `100m` and `Memory` should be `100Mi`.

   d.) Define an environment variable named `GET_HOSTS_FROM` and its value should be `dns`.

   e.) Container port should be Redis default port i.e `6379`.
4. Create another service named `redis-slave`. It should use Redis default port i.e `6379`.
5. Create another service named `redis-follower`. Port and targetPort should be Redis default port i.e `6379`. Its selector `app` should be `redis-slave`.

`FRONT END TIER`

1. Create a deployment named `frontend`.

   a.) Replicas count should be `3`.

   b.) Container name should be `php-redis-xfusion` and it should use `gcr.io/google-samples/gb-frontend@sha256:a908df8486ff66f2c4daa0d3d8a2fa09846a1fc8efd65649c0109695c7c5cbff` image.

   c.) Request resources as `CPU` should be `100m` and `Memory` should be `100Mi`.

   d.) Define an environment variable named as `GET_HOSTS_FROM` and its value should be `dns`.

   e.) Container port should be `80`.
2. Create a service named `frontend`. Its `type` should be `NodePort`, port should be `80` and its `nodePort` should be `30009`.

Finally, you can check the `guestbook app` by clicking on `App` button.

`You can use any labels as per your choice.`

`Note:` The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

The application has a Redis master, two Redis replicas, and three web frontends. Each tier has a Service whose selector matches its Pods. The `redis-slave` and `redis-follower` Services both select the two replica Pods; the additional name does not require another Deployment. The frontend Service exposes the page through NodePort `30009`.

### 📝 Step 1: Create the Guestbook Manifest

```bash
vi guestbook.yaml
```

Add the seven resources below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-master
  template:
    metadata:
      labels:
        app: redis-master
    spec:
      containers:
        - name: master-redis-xfusion
          image: redis
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
          ports:
            - containerPort: 6379

---
apiVersion: v1
kind: Service
metadata:
  name: redis-master
spec:
  selector:
    app: redis-master
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-slave
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis-slave
  template:
    metadata:
      labels:
        app: redis-slave
    spec:
      containers:
        - name: slave-redis-xfusion
          image: gcr.io/google_samples/gb-redisslave:v3
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
          env:
            - name: GET_HOSTS_FROM
              value: dns
          ports:
            - containerPort: 6379

---
apiVersion: v1
kind: Service
metadata:
  name: redis-slave
spec:
  selector:
    app: redis-slave
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379

---
apiVersion: v1
kind: Service
metadata:
  name: redis-follower
spec:
  selector:
    app: redis-slave
  ports:
    - protocol: TCP
      port: 6379
      targetPort: 6379

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: php-redis-xfusion
          image: gcr.io/google-samples/gb-frontend@sha256:a908df8486ff66f2c4daa0d3d8a2fa09846a1fc8efd65649c0109695c7c5cbff
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
          env:
            - name: GET_HOSTS_FROM
              value: dns
          ports:
            - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30009
```

> **Why:** `vi` opens a text editor on the jump host so the seven related objects can live in one YAML file. Each `---` starts a new resource. A Deployment keeps the requested number of Pods running; its `selector.matchLabels` must match the labels in its Pod template. A Service routes traffic to Pods whose labels match its own selector.
>
> `redis-master` requests one Pod; `redis-slave` requests two; `frontend` requests three. Each container requests `100m` CPU, meaning one tenth of a core, and `100Mi` memory, meaning 100 mebibytes. Requests help the scheduler place the Pods; they are not resource limits. Redis listens on port `6379`, and the frontend listens on port `80`.
>
> `GET_HOSTS_FROM=dns` tells the supplied replica and frontend images to use DNS names for their dependencies. Both `redis-slave` and `redis-follower` Services select `app: redis-slave`, so they provide two names for the same replica Pods. Their default Service type is `ClusterIP`, which keeps Redis accessible inside the cluster. The frontend uses `NodePort` so clients can reach its Service port `80` through node port `30009`. The frontend image is pinned to the exact digest required by the challenge.

### 🚀 Step 2: Apply the Manifest

```bash
kubectl apply -f guestbook.yaml
```

The lab created three Deployments and four Services:

```text
deployment.apps/redis-master created
service/redis-master created
deployment.apps/redis-slave created
service/redis-slave created
service/redis-follower created
deployment.apps/frontend created
service/frontend created
```

> **Why:** `kubectl apply` sends the manifest to the Kubernetes API. The `-f` option names the file containing the resources. Kubernetes then creates the Redis and frontend Pods and their Services from those declarations.

### ⏳ Step 3: Wait for All Deployments

```bash
kubectl rollout status deploy
```

All three rollouts completed:

```text
deployment "frontend" successfully rolled out
deployment "redis-master" successfully rolled out
deployment "redis-slave" successfully rolled out
```

> **Why:** `kubectl rollout status` waits for Deployments to have their requested available replicas. `deploy` is the short name for Deployment; without a specific Deployment name, this command reports the rollouts of Deployments in the current namespace.

### 🔍 Step 4: Check the Pods and Services

```bash
kubectl get pods
kubectl get service
```

The six application Pods were all ready and running: three `frontend`, one `redis-master`, and two `redis-slave`. The four application Services appeared as follows:

```text
NAME             TYPE        PORT(S)
frontend         NodePort    80:30009/TCP
redis-follower   ClusterIP   6379/TCP
redis-master     ClusterIP   6379/TCP
redis-slave      ClusterIP   6379/TCP
```

> **Why:** `kubectl get pods` confirms that all six containers reached `1/1 Running`. `kubectl get service` confirms the names, internal Redis ports, and the frontend's required `NodePort`. The `kubernetes` Service that also appears in the default namespace is a cluster-provided Service, not part of this guestbook application.

### ✅ Step 5: Verify the Guestbook Page

```bash
curl http://localhost:30009/
```

The response contained the Guestbook HTML page, including `<title>Guestbook</title>` and `<h2>Guestbook</h2>`. The lab's **App** button provides another way to open the published frontend.

> **Why:** `curl` requests the application through the node's port `30009`. Receiving the HTML page confirms that the frontend NodePort reaches a running web Pod and that the guestbook page is being served. This verifies page availability; the provided output does not by itself test saving and reading a guest entry.

## Best Practices

- **Keep labels aligned with selectors.** A Service can send traffic only to Pods selected by its labels. In this lab, both read Services intentionally select `app: redis-slave`.
- **Use internal Services for Redis.** The Redis Services use `ClusterIP` so the application can reach them by stable cluster DNS names without exposing Redis through a node port.
- **Pin application images for reproducibility.** The frontend's digest identifies exact image content. In production, pin the Redis image to a tested version or digest too.
- **Test the actual user path.** Pod readiness and Service existence are useful checks, while the NodePort request verifies that the browser-facing page responds.
- **Distinguish requests from limits.** The `100m` and `100Mi` values request scheduler capacity; they do not cap runtime usage.

### 📚 Official Documentation

- [Deploying a PHP Guestbook with Redis](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services and NodePort](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
