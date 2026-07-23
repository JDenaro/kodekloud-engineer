# Task 08: Schedule Cronjobs in Kubernetes

The Nautilus DevOps team is setting up recurring tasks on different schedules. Currently, they're developing scripts to be executed periodically. To kickstart the process, they're creating cron jobs in the Kubernetes cluster with placeholder commands. Follow the instructions below:

## Task Requirements

1. Create a cronjob named `xfusion`.
2. Set Its schedule to something like `*/3 * * * *`. You can set any schedule for now.
3. Name the container `cron-xfusion`.
4. Utilize the nginx image with latest tag (specify as `nginx:latest`).
5. Execute the dummy command `echo Welcome to xfusioncorp!`.
6. Ensure the restart policy is `OnFailure`.

Note: The `kubectl` utility on the jump-host has been configured to work with the Kubernetes cluster.

## Solution

A CronJob creates Jobs on a repeating schedule. Each Job then creates a Pod from the template defined in the CronJob. The manifest must therefore place the container, command, and restart policy inside `spec.jobTemplate.spec.template.spec`.

The `nginx` image normally starts the nginx web server. This task requires a short command that prints a message and exits, so the manifest overrides the image's default command with `/bin/sh -c` and the requested `echo` command. The schedule `*/3 * * * *` runs the CronJob every three minutes.

### 📝 Step 1: Create the CronJob manifest

```bash
vi xfusion-cronjob.yaml
```

Inside the editor, press `i`, enter the following YAML, press `Esc`, type `:wq`, and press `Enter`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: xfusion
spec:
  schedule: "*/3 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cron-xfusion
              image: nginx:latest
              command:
                - /bin/sh
                - -c
                - "echo Welcome to xfusioncorp!"
          restartPolicy: OnFailure
```

> **Why:** `vi` is a terminal text editor. The `i` key enters insert mode, `Esc` leaves insert mode, and `:wq` saves the file and exits. `apiVersion: batch/v1` selects the stable API for batch resources, and `kind: CronJob` identifies the resource type. `metadata.name` assigns the required CronJob name. `spec.schedule` contains the five-field cron expression; `*/3` in the minute field means every three minutes, while the remaining `*` fields allow every hour, day, month, and weekday. `jobTemplate` describes each Job that the CronJob creates. Inside the Pod template, `name` sets the required container name and `image` selects `nginx:latest` with the explicit tag.

The `command` list replaces the image's default command. `/bin/sh` starts the shell, `-c` tells the shell to execute the following string, and the final item prints `Welcome to xfusioncorp!`. `restartPolicy: OnFailure` tells Kubernetes to restart the Pod's container when the command fails; it is placed at the Pod template level because restart policy is a Pod setting.

### 🚀 Step 2: Create the CronJob

```bash
kubectl apply -f xfusion-cronjob.yaml
```

The CronJob was created successfully:

```text
thor@jump-host ~$ vi xfusion-cronjob.yaml
thor@jump-host ~$ kubectl apply -f xfusion-cronjob.yaml
cronjob.batch/xfusion created
thor@jump-host ~$
```

> **Why:** `kubectl` is the Kubernetes command-line client, and `apply` sends the desired resource configuration to the cluster. The `-f` option tells `kubectl` to read the configuration from a file, and `xfusion-cronjob.yaml` is the manifest created in the previous step. The output confirms that Kubernetes accepted the CronJob resource.

### ✅ Step 3: Verify the CronJob schedule

```bash
kubectl get cronjobs
```

The verification showed the requested CronJob and schedule:

```text
thor@jump-host ~$ kubectl get cronjobs
NAME      SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
xfusion   */3 * * * *   <none>     False     0        8s              27s
thor@jump-host ~$
```

> **Why:** `get` retrieves information about Kubernetes resources, and `cronjobs` selects all CronJobs in the current namespace. The `SCHEDULE` column confirms the three-minute schedule. `SUSPEND=False` means scheduling is enabled, `ACTIVE=0` means no Job was running at the exact time of the check, and `LAST SCHEDULE=8s` confirms that the CronJob had already created a scheduled Job recently. `TIMEZONE=<none>` means the CronJob uses the cluster's default time-zone behavior because no explicit time zone was configured.

## Best Practices

- **Use a CronJob for recurring work.** A CronJob creates a new Job according to its schedule, while a Job manages the Pod that performs one execution.
- **Keep commands explicit.** The manifest overrides the `nginx` image's default behavior so the placeholder command exits after printing the requested message.
- **Use the correct Pod-level restart policy.** `OnFailure` is valid for scheduled Job Pods and allows failed executions to be retried.
- **Quote cron expressions in YAML.** Quoting `*/3 * * * *` keeps the schedule as a single string and avoids YAML parsing problems with special characters.
- **Check scheduling separately from execution.** `kubectl get cronjobs` confirms that the CronJob is enabled and has scheduled work; the resulting Jobs and Pods can be inspected separately when troubleshooting execution.
- **Make recurring commands safe to repeat.** CronJobs can occasionally create a Job more than once or miss a schedule under certain cluster conditions, so production commands should be idempotent whenever possible.

### 📚 Official Documentation

- [Running automated tasks with a CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)
- [Kubernetes CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [`kubectl apply` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
