# Task 09: Create Countdown Job in Kubernetes

The Nautilus DevOps team is crafting jobs in the Kubernetes cluster. While they're developing actual scripts/commands, they're currently setting up templates and testing jobs with dummy commands. Please create a job template as per details given below:

## Task Requirements

1. Create a job named `countdown-nautilus`.
2. The spec template should be named `countdown-nautilus` (under metadata), and the container should be named `container-countdown-nautilus`.
3. Utilize image `debian` with latest tag (ensure to specify as `debian:latest`), and set the restart policy to `Never`.
4. Execute the command `sleep 5`.

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

A Job runs a finite task and tracks whether it completes successfully. This Job creates a Pod template with the requested name, container name, image, command, and restart policy. The `sleep 5` command keeps the container alive for five seconds and then exits successfully, allowing the Job to complete.

The restart policy is placed under `spec.template.spec` because it is a Pod setting. `Never` tells Kubernetes not to restart the container inside the same Pod if it fails; the Job controller can create a replacement Pod if the Job needs to retry according to its failure settings.

### 📝 Step 1: Create the Job manifest

```bash
vi countdown-nautilus.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: countdown-nautilus
spec:
  template:
    metadata:
      name: countdown-nautilus
    spec:
      containers:
        - name: container-countdown-nautilus
          image: debian:latest
          command:
            - sleep
            - "5"
      restartPolicy: Never
```

> **Why:** `vi` is a terminal text editor. The `i` key enters insert mode, `Esc` leaves insert mode, and `:wq` saves the file and exits. `apiVersion: batch/v1` selects the stable API for batch resources, and `kind: Job` identifies the resource type. The top-level `metadata.name` sets the Job name. The nested `spec.template.metadata.name` sets the requested Pod template name. Under the template's Pod specification, `name` defines the container name and `image` selects `debian:latest` with the explicit tag. The `command` list runs `sleep` with the argument `5`, which keeps the container running for five seconds. `restartPolicy: Never` is a valid Job Pod policy and prevents the container from being restarted within the same Pod.

### 🚀 Step 2: Create the Job

```bash
kubectl apply -f countdown-nautilus.yaml
```

The Job was created successfully:

```text
thor@jump-host ~$ vi countdown-nautilus.yaml
thor@jump-host ~$ kubectl apply -f countdown-nautilus.yaml
job.batch/countdown-nautilus created
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client, and `apply` sends the desired resource configuration to the cluster. The `-f` option tells `kubectl` to read the definition from a file, and `countdown-nautilus.yaml` is the manifest created in the previous step. The output confirms that the Job resource was accepted and created.

### ✅ Step 3: Check the Job status

```bash
kubectl get job
```

The lab session showed the newly created Job while its Pod was still running:

```text
thor@jump-host ~$ kubectl get job
NAME                 STATUS    COMPLETIONS   DURATION   AGE
countdown-nautilus   Running   0/1           7s         7s
thor@jump-host ~$
thor@jump-host ~$ kubectl get job
NAME                 STATUS     COMPLETIONS   DURATION   AGE
countdown-nautilus   Complete   1/1           11s        30s
thor@jump-host ~$
```

The first check showed `Running` with `0/1` because the Job had not yet recorded a completed execution. The second check showed `Complete` with `1/1`, confirming that the container ran `sleep 5` and exited successfully. Image-pull time and scheduling made the total Job duration longer than five seconds, which is normal.

> **Why:** `get` retrieves information about Kubernetes resources, and `job` selects Job resources. `STATUS=Running` means the Job was active during the first check, while `COMPLETIONS=0/1` meant that one successful completion was required and none had been recorded yet. The later `STATUS=Complete` and `COMPLETIONS=1/1` confirmed successful execution. `DURATION` and `AGE` show how long the Job had existed and been active.

## Best Practices

- **Use a Job for finite work.** Jobs are designed for commands that should run to completion, unlike Deployments and ReplicaSets, which manage long-running application Pods.
- **Set a valid Job restart policy.** Job Pod templates support `Never` and `OnFailure`; this task explicitly requires `Never`.
- **Keep the command separate from the image.** The `command` field makes the task's behavior explicit instead of relying on the image's default entrypoint.
- **Use explicit image tags.** The task requires `debian:latest`; production jobs should generally use a fixed version for repeatable execution.
- **Allow for startup time.** A short command such as `sleep 5` may still take longer than five seconds to reach `Complete` because the image may need to be pulled and the Pod scheduled.
- **Inspect completions, not only status.** A successful Job should eventually report `Complete` with `1/1` in the `COMPLETIONS` column.

### 📚 Official Documentation

- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [`kubectl get` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/)
