# Day 63: Deploy Iron Gallery App on Kubernetes

There is an iron gallery app that the Nautilus DevOps team was developing. They have recently customized the app and are going to deploy the same on the Kubernetes cluster. Below you can find more details:

## Specific Requirements:

1. Create a namespace `iron-namespace-datacenter`.
2. Create a deployment `iron-gallery-deployment-datacenter` for iron gallery under the same namespace you created.
   - Labels `run` should be `iron-gallery`.
   - Replicas count should be `1`.
   - Selector's `matchLabels run` should be `iron-gallery`.
   - Template labels `run` should be `iron-gallery` under metadata.
   - The container should be named as `iron-gallery-container-datacenter`, use `kodekloud/irongallery:2.0` image (use exact image name / tag).
   - Resources limits for memory should be `100Mi` and for CPU should be `50m`.
   - First `volumeMount` name should be `config`, its `mountPath` should be `/usr/share/nginx/html/data`.
   - Second `volumeMount` name should be `images`, its `mountPath` should be `/usr/share/nginx/html/uploads`.
   - First volume name should be `config` and give it `emptyDir` and second volume name should be `images`, also give it `emptyDir`.
3. Create a deployment `iron-db-deployment-datacenter` for iron db under the same namespace.
   - Labels `db` should be `mariadb`.
   - Replicas count should be `1`.
   - Selector's `matchLabels db` should be `mariadb`.
   - Template labels `db` should be `mariadb` under metadata.
   - The container name should be `iron-db-container-datacenter`, use `kodekloud/irondb:2.0` image (use exact image name / tag).
   - Define environment, set `MYSQL_DATABASE` its value should be `database_host`, set `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD` value should be with some complex passwords for DB connections, and `MYSQL_USER` value should be any custom user (except root).
   - Volume mount name should be `db` and its `mountPath` should be `/var/lib/mysql`. Volume name should be `db` and give it an `emptyDir`.
4. Create a service for iron db which should be named `iron-db-service-datacenter` under the same namespace. Configure spec as selector's `db` should be `mariadb`. Protocol should be `TCP`, `port` and `targetPort` should be `3306` and its type should be `ClusterIP`.
5. Create a service for iron gallery which should be named `iron-gallery-service-datacenter` under the same namespace. Configure spec as selector's `run` should be `iron-gallery`. Protocol should be `TCP`, `port` and `targetPort` should be `80`, `nodePort` should be `32678` and its type should be `NodePort`.

`Note:`

We don't need to make connection b/w database and front-end now, if the installation page is coming up it should be enough for now.

The `kubectl` utility on the `jump-host` has been configured to work with the Kubernetes cluster.

## Solution

This deployment uses a namespace to group the application resources, Deployments to keep the gallery and database Pods running, temporary `emptyDir` volumes for the lab data, and two different Service types. MariaDB receives a private `ClusterIP` Service because it is a backend component, while the gallery receives a `NodePort` Service so that the installation page can be reached from outside the cluster.

### 📁 Step 1: Create the Namespace

```bash
kubectl create namespace iron-namespace-datacenter
```

Kubernetes created the namespace:

```text
namespace/iron-namespace-datacenter created
```

> **Why:** `kubectl create namespace` creates a logical boundary named `iron-namespace-datacenter`. Namespaces group related namespaced resources and allow resources with similar names to coexist in different environments. Every Deployment and Service in this challenge explicitly uses this namespace.

### 📝 Step 2: Create the Application Manifest

```bash
vi iron-gallery-datacenter.yaml
```

Add the following configuration. Replace the two password placeholders with complex temporary values before applying it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-gallery-deployment-datacenter
  namespace: iron-namespace-datacenter
  labels:
    run: iron-gallery
spec:
  replicas: 1
  selector:
    matchLabels:
      run: iron-gallery
  template:
    metadata:
      labels:
        run: iron-gallery
    spec:
      containers:
        - name: iron-gallery-container-datacenter
          image: kodekloud/irongallery:2.0
          resources:
            limits:
              memory: 100Mi
              cpu: 50m
          volumeMounts:
            - name: config
              mountPath: /usr/share/nginx/html/data
            - name: images
              mountPath: /usr/share/nginx/html/uploads
      volumes:
        - name: config
          emptyDir: {}
        - name: images
          emptyDir: {}

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iron-db-deployment-datacenter
  namespace: iron-namespace-datacenter
  labels:
    db: mariadb
spec:
  replicas: 1
  selector:
    matchLabels:
      db: mariadb
  template:
    metadata:
      labels:
        db: mariadb
    spec:
      containers:
        - name: iron-db-container-datacenter
          image: kodekloud/irondb:2.0
          env:
            - name: MYSQL_DATABASE
              value: database_host
            - name: MYSQL_ROOT_PASSWORD
              value: "REPLACE_WITH_A_COMPLEX_ROOT_PASSWORD"
            - name: MYSQL_PASSWORD
              value: "REPLACE_WITH_A_COMPLEX_USER_PASSWORD"
            - name: MYSQL_USER
              value: datacenter_user
          volumeMounts:
            - name: db
              mountPath: /var/lib/mysql
      volumes:
        - name: db
          emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: iron-db-service-datacenter
  namespace: iron-namespace-datacenter
spec:
  type: ClusterIP
  selector:
    db: mariadb
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306

---
apiVersion: v1
kind: Service
metadata:
  name: iron-gallery-service-datacenter
  namespace: iron-namespace-datacenter
spec:
  type: NodePort
  selector:
    run: iron-gallery
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 32678
```

> **Why:** `vi` creates one readable multi-document YAML file. Each `---` separator starts another Kubernetes object, allowing both Deployments and both Services to be managed together. `apiVersion: apps/v1` and `kind: Deployment` define controllers that maintain the requested single replica. Each Deployment selector exactly matches its Pod-template label, which is how the controller identifies the Pods it owns. The exact image tags make the deployed versions explicit.
>
> The gallery container has a memory limit of `100Mi` (100 mebibytes) and a CPU limit of `50m` (50 millicores, or 5% of one CPU core). Its `config` and `images` volume mounts match Pod-level volumes with the same names. An `emptyDir` volume begins empty when the Pod is assigned to a node, is shared by containers in that Pod, and is deleted when the Pod is removed from the node. The database follows the same pattern for `/var/lib/mysql`; this is suitable for a temporary lab, but not durable database storage.
>
> The MariaDB variables initialize the requested database and non-root user. The real lab passwords are represented by placeholders so credentials are not committed to source control. In a production manifest, they should come from a Kubernetes Secret rather than literal `value` fields.

### 🚀 Step 3: Create the Deployments and Services

```bash
kubectl apply -f iron-gallery-datacenter.yaml
```

The manifest created both Deployments and both Services:

```text
deployment.apps/iron-gallery-deployment-datacenter created
deployment.apps/iron-db-deployment-datacenter created
service/iron-db-service-datacenter created
service/iron-gallery-service-datacenter created
```

> **Why:** `kubectl apply` sends the declarative resource definitions to the Kubernetes API. The `-f` option tells `kubectl` to read them from `iron-gallery-datacenter.yaml`. Since every object declares `namespace: iron-namespace-datacenter`, all four application resources are created in the required namespace.

### ⏳ Step 4: Wait for Both Deployments

```bash
kubectl rollout status deployment/iron-gallery-deployment-datacenter -n iron-namespace-datacenter
kubectl rollout status deployment/iron-db-deployment-datacenter -n iron-namespace-datacenter
```

Both rollouts completed successfully:

```text
deployment "iron-gallery-deployment-datacenter" successfully rolled out
deployment "iron-db-deployment-datacenter" successfully rolled out
```

> **Why:** `kubectl rollout status` waits for each Deployment to finish creating its available replica. The resource is written as `deployment/<name>`, and `-n iron-namespace-datacenter` tells `kubectl` to inspect the namespace used by this application.

### 🔍 Step 5: Verify the Pods

```bash
kubectl get pods -n iron-namespace-datacenter
```

The gallery and database Pods must both report `1/1` under `READY` and `Running` under `STATUS`.

> **Why:** `kubectl get pods` provides a direct, readable view of Pod readiness. `1/1` means the only container in each Pod is ready, while `Running` confirms that Kubernetes successfully pulled the required image and started the container.

### 🌐 Step 6: Verify the ClusterIP and NodePort Services

```bash
kubectl get services -n iron-namespace-datacenter
kubectl describe service iron-db-service-datacenter -n iron-namespace-datacenter
kubectl describe service iron-gallery-service-datacenter -n iron-namespace-datacenter
```

The service list must show:

```text
iron-db-service-datacenter        ClusterIP   ...   3306/TCP
iron-gallery-service-datacenter   NodePort    ...   80:32678/TCP
```

Both descriptions must show a Pod IP under `Endpoints` rather than `<none>`.

> **Why:** The database uses `ClusterIP` because it is an internal backend. A `ClusterIP` Service gives MariaDB a stable virtual IP and DNS name that other workloads inside the cluster can use on port `3306`, without opening a port on the nodes. This reduces unnecessary exposure of the database.
>
> The gallery uses `NodePort` because users and the KodeKloud Website button must reach the frontend from outside the cluster. `NodePort` still creates an internal ClusterIP, but it additionally listens on port `32678` on every node and forwards that traffic through Service port `80` to target port `80` on the selected gallery Pod. The `run: iron-gallery` and `db: mariadb` selectors connect each Service to the correct Pods. `kubectl describe service` exposes those selected endpoints in a beginner-friendly format.

The traffic paths are:

```text
Cluster workload -> iron-db-service-datacenter:3306 -> MariaDB Pod:3306
External client -> Node IP:32678 -> Gallery Service:80 -> Gallery Pod:80
```

### ✅ Step 7: Verify the Installation Page

```bash
curl http://localhost:32678/
```

The command returned the Iron Gallery installation page. The same page was available through the KodeKloud **Website** button, which satisfied the challenge; connecting the frontend to MariaDB was explicitly outside the scope of this lab.

> **Why:** `curl` sends an HTTP request to NodePort `32678` on the local cluster node. Receiving the installation-page HTML proves that the node accepts traffic on the requested port, the NodePort Service forwards it to an endpoint, and the Iron Gallery container responds on port `80`.

## Best Practices

- **Keep databases private.** Use `ClusterIP` for database backends unless there is a justified need for direct external access. Application workloads can use the Service DNS name inside the cluster.
- **Use NodePort mainly for simple exposure and labs.** Production environments commonly place an Ingress, Gateway, or `LoadBalancer` Service in front of applications instead of requiring clients to know node addresses and high-numbered ports.
- **Store credentials in Secrets.** Literal environment-variable values are easy to expose through manifests and version control. Reference Secret keys with `valueFrom.secretKeyRef` in real deployments.
- **Use persistent storage for databases.** `emptyDir` data disappears with the Pod. Stateful database workloads should use PersistentVolumeClaims and an appropriate storage class.
- **Set both requests and limits in production.** This challenge requires limits only. Requests additionally help the scheduler reserve enough CPU and memory for predictable operation.
- **Keep selectors and Pod labels aligned.** A Deployment cannot manage the intended Pods, and a Service has no endpoints, when its selector does not match the Pod-template labels.
- **Use immutable image references where appropriate.** Exact tags are clearer than implicit tags, while digest pinning offers even stronger reproducibility for controlled production deployments.

### 📚 Official Documentation

- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Services, ClusterIP, and NodePort](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Volumes and emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- [Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Define Environment Variables for a Container](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
