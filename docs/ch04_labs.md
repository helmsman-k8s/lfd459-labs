# Chapter 4 - Design

## Lab Overview

In this chapter you will review CNI network plugin concepts, work with Jobs and CronJobs for time-limited workloads, use label selectors to manage pods, set CPU and memory resource limits, and explore initContainers and Custom Resource Definitions.

---

## Exercise 4.1: Planning the Deployment

### Quick Understanding of Network Plugins

While application developers don't configure cluster networking, understanding the CNI plugin in use helps when troubleshooting and designing network policies.

1. Connect to the **controller** node. Verify the CNI plugin is active by reading the controller manager startup logs. Use Tab completion for the pod name.

    ```bash
    kubectl -n kube-system logs kube-controller-manager-controller
    ```

    Look for lines referencing the network plugin initialization.

2. Check the kube-proxy logs to see which proxy mode is active.

    ```bash
    kubectl -n kube-system logs kube-proxy-<TAB>
    ```

    !!! note "Our CNI: Calico"
        This cluster uses **Calico** as the CNI plugin. Calico provides a layer 3 network using BGP by default and fully supports NetworkPolicies.

3. Review the properties of common CNI plugins using the links below, then answer the questions that follow.

    - **Flannel**: <https://github.com/coreos/flannel>
    - **Calico**: <https://docs.projectcalico.org/>
    - **Cilium**: <https://cilium.io/>
    - **Kube Router**: <https://www.kube-router.io>

    Answer the following questions:

    - Which plugins support vxlans?
    - Which are layer 2 plugins?
    - Which are layer 3?
    - Which support network policies?
    - Which can encrypt all TCP and UDP traffic?

??? success "Plugin Answers"
    - **vxlans**: Canal, Calico, Flannel, Weave Net, Cilium
    - **Layer 2**: Canal, Weave Net
    - **Layer 3**: Calico, Romana, Kube Router
    - **Network policies**: Calico, Canal, Kube Router, Weave Net, Cilium
    - **Encrypt TCP/UDP**: Calico, Weave Net, Cilium

    !!! note
        Flannel does **not** support network policies natively. Calico, which this cluster uses, does. Keep this in mind in Chapter 6.

### Multi-Container Pod Considerations

Consider the following questions based on what you learned in this chapter:

1. Which deployment method allows the most flexibility - one app per pod or multiple?
2. Which allows the most granular scalability?
3. Which has the best inter-container performance?
4. How many IP addresses are assigned per pod?
5. What are some ways containers can communicate within the same pod?
6. What are some reasons to use multiple containers per pod?

??? success "Multi-Pod Answers"
    1. One per pod - most flexible
    2. One per pod - most granular scalability
    3. Multiple per pod - best inter-container performance
    4. One IP per pod
    5. IPC, loopback interface, shared filesystem
    6. Lean containers may lack logging or other features. Adding sidecar, ambassador, or adapter containers provides that functionality without modifying the primary container.

---

## Exercise 4.2: Create a Job

Jobs run a container a set number of times to completion, rather than keeping it running continuously.

1. Move into the chapter lab directory.

    ```bash
    cd ~/lfd459/ch04-design
    ls
    ```

2. Review and create the basic job definition.

    ```bash
    cat job.yaml
    ```

    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: sleepy
    spec:
      template:
        spec:
          containers:
          - name: resting
            image: busybox
            command: ["/bin/sleep"]
            args: ["3"]
          restartPolicy: Never
    ```

    ```bash
    kubectl create -f job.yaml
    ```

    ```
    job.batch/sleepy created
    ```

3. Verify the job completes successfully. Check while it is running, then again after it finishes.

    ```bash
    kubectl get job
    ```

    ```
    NAME     COMPLETIONS   DURATION   AGE
    sleepy   0/1           3s         3s
    ```

    ```bash
    kubectl get job
    ```

    ```
    NAME     COMPLETIONS   DURATION   AGE
    sleepy   1/1           7s         11s
    ```

4. Describe the job to see its full details including parallelism, completions, and pod statuses.

    ```bash
    kubectl describe jobs.batch sleepy
    ```

5. Inspect the default job parameters in YAML output. Note `backoffLimit: 6`, `completions: 1`, `parallelism: 1`.

    ```bash
    kubectl get jobs.batch sleepy -o yaml | grep -A5 "^spec:"
    ```

6. Delete the job.

    ```bash
    kubectl delete jobs.batch sleepy
    ```

    ```
    job.batch "sleepy" deleted
    ```

7. Edit `job.yaml` and add `completions: 5` to run the job five times sequentially.

    ```bash
    vim job.yaml
    ```

    ```yaml
    spec:
      completions: 5        # <-- add this line
      template:
    ```

    !!! tip
        If you prefer not to edit manually, use the pre-built version: `kubectl create -f job-v2.yaml`

8. Create the job again and watch the completions count up.

    ```bash
    kubectl create -f job.yaml
    kubectl get jobs.batch
    ```

    ```
    NAME     COMPLETIONS   DURATION   AGE
    sleepy   0/5           5s         5s
    ```

9. Once all 5 complete, delete the job.

    ```bash
    kubectl get jobs.batch
    ```

    ```
    NAME     COMPLETIONS   DURATION   AGE
    sleepy   5/5           26s        10m
    ```

    ```bash
    kubectl delete jobs.batch sleepy
    ```

10. Edit `job.yaml` and add `parallelism: 2` so two pods run at the same time.

    ```bash
    vim job.yaml
    ```

    ```yaml
    spec:
      completions: 5
      parallelism: 2        # <-- add this line
      template:
    ```

    !!! tip
        Pre-built version available: `kubectl create -f job-v3.yaml`

11. Create the job and verify two pods run simultaneously.

    ```bash
    kubectl create -f job.yaml
    kubectl get pods
    ```

    ```
    NAME             READY   STATUS    RESTARTS   AGE
    sleepy-8xwpc     1/1     Running   0          5s
    sleepy-xjqnf     1/1     Running   0          5s
    ...
    ```

12. Edit `job.yaml` again and add `activeDeadlineSeconds: 15` to forcibly stop the job after 15 seconds.

    ```bash
    vim job.yaml
    ```

    ```yaml
    spec:
      completions: 5
      parallelism: 2
      activeDeadlineSeconds: 15   # <-- add this line
      template:
    ```

    !!! tip
        Pre-built version available: `kubectl create -f job-v4.yaml`

13. Delete and recreate the job. It will stop after 15 seconds regardless of how many completions remain.

    ```bash
    kubectl delete jobs.batch sleepy
    kubectl create -f job.yaml
    ```

    Wait ~20 seconds, then verify the job stopped early.

    ```bash
    kubectl get jobs
    ```

    ```
    NAME     COMPLETIONS   DURATION   AGE
    sleepy   4/5           16s        16s
    ```

14. Check the status message in the job YAML - it should show `DeadlineExceeded`.

    ```bash
    kubectl get job sleepy -o yaml | grep -A8 "^status:"
    ```

    ```
    status:
      conditions:
      - message: Job was active longer than specified deadline
        reason: DeadlineExceeded
        status: "True"
        type: Failed
      succeeded: 4
    ```

15. Delete the job.

    ```bash
    kubectl delete jobs.batch sleepy
    ```

---

## Exercise 4.3: Create a CronJob

A CronJob creates Jobs on a recurring schedule using standard Linux cron syntax.

1. Review the provided `cronjob.yaml` file. It runs the sleep container every 2 minutes.

    ```bash
    cat cronjob.yaml
    ```

    ```yaml
    apiVersion: batch/v1
    kind: CronJob
    metadata:
      name: sleepy
    spec:
      schedule: "*/2 * * * *"
      jobTemplate:
        spec:
          template:
            spec:
              containers:
              - name: resting
                image: busybox
                command: ["/bin/sleep"]
                args: ["3"]
              restartPolicy: Never
    ```

    !!! warning "batch/v1beta1 is removed"
        A file named `cron-job.yaml` is also in your directory - it uses `apiVersion: batch/v1beta1` which was **removed in Kubernetes 1.25**. Do not use it. Always use `batch/v1` for CronJobs on this cluster.

2. Create the CronJob and monitor it. The first job will not run for up to 2 minutes.

    ```bash
    kubectl create -f cronjob.yaml
    ```

    ```
    cronjob.batch/sleepy created
    ```

    ```bash
    kubectl get cronjobs.batch
    ```

    ```
    NAME     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
    sleepy   */2 * * * *   False     0        <none>          8s
    ```

3. After 2 minutes, verify a Job was created by the CronJob.

    ```bash
    kubectl get cronjobs.batch
    ```

    ```
    NAME     SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
    sleepy   */2 * * * *   False     0        21s             2m1s
    ```

    ```bash
    kubectl get jobs.batch
    ```

    ```
    NAME                COMPLETIONS   DURATION   AGE
    sleepy-1539722040   1/1           5s         18s
    ```

    Let it run for a few more cycles and you will see multiple completed Jobs listed.

4. Edit `cronjob.yaml` to make the sleep command run for 30 seconds and add `activeDeadlineSeconds: 10` inside the job template, so that each job is forcibly killed after 10 seconds.

    ```bash
    vim cronjob.yaml
    ```

    Modify the `jobTemplate.spec.template.spec` section:

    ```yaml
    jobTemplate:
      spec:
        template:
          spec:
            activeDeadlineSeconds: 10   # <-- add this line
            containers:
            - name: resting
              image: busybox
              command: ["/bin/sleep"]
              args: ["30"]              # <-- change from "3" to "30"
            restartPolicy: Never
    ```

5. Delete and recreate the CronJob. Wait 2 minutes, then observe that jobs are created but never complete - they are killed by the deadline.

    ```bash
    kubectl delete cronjobs.batch sleepy
    kubectl create -f cronjob.yaml
    ```

    !!! tip
        Pre-built version available: `kubectl create -f cronjob-v2.yaml`

    Wait ~2 minutes, then check:

    ```bash
    kubectl get jobs
    ```

    ```
    NAME                COMPLETIONS   DURATION   AGE
    sleepy-1539723240   0/1           61s        61s
    ```

6. Delete the CronJob and all its associated jobs.

    ```bash
    kubectl delete cronjobs.batch sleepy
    ```

    ```
    cronjob.batch "sleepy" deleted
    ```

---

## Exercise 4.4: Using Labels

Labels are key-value pairs attached to objects. Selectors use them to filter and manage groups of objects.

1. Create a new `design2` deployment.

    ```bash
    kubectl create deployment design2 --image=nginx
    ```

    ```
    deployment.apps/design2 created
    ```

2. View the deployment with `-o wide` and note the `SELECTOR` column.

    ```bash
    kubectl get deployments.apps design2 -o wide
    ```

    ```
    NAME      READY   UP-TO-DATE   AVAILABLE   ...   SELECTOR
    design2   1/1     1            1           ...   app=design2
    ```

3. Use the `-l` option to list only pods matching the deployment selector.

    ```bash
    kubectl get -l app=design2 pod
    ```

    ```
    NAME                       READY   STATUS    RESTARTS   AGE
    design2-766d48574f-5w274   1/1     Running   0          3m
    ```

4. View the pod labels in full YAML output using `--selector`.

    ```bash
    kubectl get --selector app=design2 pod -o yaml | grep -A4 "labels:"
    ```

    You should see both `app: design2` and `pod-template-hash` labels.

5. Edit the pod's `app` label and change it to your favourite colour (e.g. `orange`).

    ```bash
    kubectl edit pod design2-766d48574f-5w274
    ```

    Find and change:

    ```yaml
    labels:
      app: orange    # <-- change from design2
      pod-template-hash: 766d48574f
    ```

6. Immediately check how many pods the deployment now manages.

    ```bash
    kubectl get pods | grep design2
    ```

    ```
    design2-766d48574f-5w274   1/1   Running   0   82s
    design2-766d48574f-xttgg   1/1   Running   0   2m12s
    ```

    There are now **two** pods. The deployment noticed the edited pod no longer matched its selector (`app=design2`) and created a replacement. The edited pod is now orphaned - no controller manages it.

7. Delete the deployment.

    ```bash
    kubectl delete deploy design2
    ```

    ```
    deployment.apps "design2" deleted
    ```

8. Check for remaining pods. The orphaned pod (with the colour label) still exists - the deployment deletion only removed pods it was managing.

    ```bash
    kubectl get pods | grep design2
    ```

    ```
    design2-766d48574f-5w274   1/1   Running   0   38m
    ```

9. Delete the orphaned pod using the colour label you assigned.

    ```bash
    kubectl delete pod -l app=orange
    ```

    ```
    pod "design2-766d48574f-5w274" deleted
    ```

---

## Exercise 4.5: Setting Pod Resource Limits

Resource requests tell the scheduler how much CPU/memory a pod needs. Limits cap what it can use.

1. Review and create the `stress.yaml` deployment. It runs the `vish/stress` image which deliberately consumes CPU and memory.

    ```bash
    cat stress.yaml
    kubectl create -f stress.yaml
    ```

2. On the **controller**, run `top` to see the stress process consuming CPU. Use `q` to exit.

    ```bash
    top
    ```

    Then open a separate terminal on **worker1** and run the same:

    ```bash
    top
    ```

3. Delete the deployment, then edit `stress.yaml` to add resource limits and requests.

    ```bash
    kubectl delete -f stress.yaml
    vim stress.yaml
    ```

    Add the `resources` block under the container name:

    ```yaml
    containers:
    - image: vish/stress
      name: stressmeout
      resources:
        limits:
          cpu: "1"
          memory: "1Gi"
        requests:
          cpu: "0.5"
          memory: "500Mi"
      args:
      - -cpus
      - "2"
      - -mem-total
      - "1950Mi"
    ```

    !!! tip
        Pre-built version available: `kubectl create -f stress-v2.yaml`

4. Create the deployment and check the pod status. Because the stress command requests more memory than the limit, the container will be OOMKilled.

    ```bash
    kubectl create -f stress.yaml
    kubectl get pod
    ```

    ```
    NAME                           READY   STATUS      RESTARTS   AGE
    stressmeout-7fbbbcc887-v9kvb   0/1     OOMKilled   2          32s
    ```

5. Delete the deployment and edit `stress.yaml` again. This time:

    - Increase the memory limit to `2Gi` (enough for the stress workload)
    - Uncomment and set the `nodeSelector` to pin the pod to **worker1** (to protect the control plane)

    ```bash
    kubectl delete -f stress.yaml
    vim stress.yaml
    ```

    ```yaml
    spec:
      nodeSelector:
        kubernetes.io/hostname: worker1    # <-- uncomment and set to worker1
      containers:
    ...
      resources:
        limits:
          cpu: "2"
          memory: "2Gi"
        requests:
          cpu: "0.5"
          memory: "500Mi"
    ```

    !!! tip
        Pre-built version available: `kubectl create -f stress-v3.yaml`

6. Create the deployment and verify the pod runs on `worker1` without OOMKilling.

    ```bash
    kubectl create -f stress.yaml
    kubectl get pod -o wide
    ```

    ```
    NAME                           READY   STATUS    NODE
    stressmeout-...                1/1     Running   worker1
    ```

7. Use `kubectl describe node worker1` to view how much of worker1's CPU and memory is now being consumed.

    ```bash
    kubectl describe node worker1 | grep -A8 "Allocated resources"
    ```

8. Try editing the limits to values higher than the node has available. Observe how the scheduler handles an unschedulable pod.

9. Delete the deployment when done.

    ```bash
    kubectl delete deploy stressmeout
    ```

---

## Exercise 4.6: Simple initContainer

An `initContainer` runs to completion before the main container starts. If it fails, the pod never becomes ready.

1. Review the provided `init-tester.yaml`. The initContainer runs `/bin/false` which always exits with a non-zero code.

    ```bash
    cat init-tester.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: init-tester
      labels:
        app: inittest
    spec:
      containers:
      - name: webservice
        image: nginx
      initContainers:
      - name: failed
        image: busybox
        command: [/bin/false]
    ```

2. Create the pod and observe the failure.

    ```bash
    kubectl create -f init-tester.yaml
    kubectl get pod init-tester
    ```

    ```
    NAME           READY   STATUS       RESTARTS   AGE
    init-tester    0/1     Init:Error   2          30s
    ```

    Use `describe` to read the init container failure events.

    ```bash
    kubectl describe pod init-tester
    ```

3. Delete the pod, fix the command from `/bin/false` to `/bin/true`, and recreate it.

    ```bash
    kubectl delete pod init-tester
    vim init-tester.yaml
    ```

    Change:

    ```yaml
    command: [/bin/true]
    ```

    !!! tip
        Pre-built version available: `kubectl create -f init-tester-v2.yaml`

4. Create the pod again and verify the nginx webservice container starts once the initContainer succeeds.

    ```bash
    kubectl create -f init-tester.yaml
    kubectl get pod init-tester
    ```

    ```
    NAME           READY   STATUS    RESTARTS   AGE
    init-tester    1/1     Running   0          10s
    ```

5. Delete the pod when done.

    ```bash
    kubectl delete pod init-tester
    ```

---

## Exercise 4.7: Exploring Custom Resource Definitions

CRDs extend the Kubernetes API with new resource types.

1. View all CRDs currently installed in the cluster.

    ```bash
    kubectl get crd
    ```

    With Calico as the CNI, you will see Calico-related CRDs (e.g. `caliconetworkpolicies`, `ippools`, `felixconfigurations`). If additional operators are installed you may see more.

2. Pick one of the CRDs listed and inspect its full definition.

    ```bash
    kubectl get crd <crd-name> -o yaml
    ```

    Note the `group`, `kind`, `plural`, `scope`, and available `versions`.

3. Try querying for resources of that custom type. Most will return empty unless instances have been created.

    ```bash
    kubectl get <kind-from-crd>
    ```

    ```
    No resources found
    ```

4. Explore additional CRDs in the cluster. Note that Kubernetes itself ships with several built-in CRDs for features like autoscaling and flow control.

    ```bash
    kubectl get crd | wc -l
    ```

---

## Exercise 4.8: Domain Review

!!! important
    Source pages and content in this review can change at any time. Always check the current version before your exam.

Revisit the CKAD curriculum and locate topics covered in this chapter:

- Understand Jobs and CronJobs
- Understanding and defining resource requirements, limits and quotas
- Understand multi-container Pod design patterns (sidecar, init, and others)
- Discover and use resources that extend Kubernetes (CRD)

- Create a pod using `design-review1.yaml`. Examine the resource limits and requests.

    ```bash
    kubectl create -f design-review1.yaml
    ```

- Determine the CPU and memory resource requirements of `design-pod1`.
- Edit the pod resource requirements such that the CPU limit is exactly twice the amount requested. (Hint: current limit is `2.22`, request is `0.3` - subtract `0.22` from the limit to make it exactly `2x` the request.)
- Increase the memory limit until the pod achieves `Running` status and holds it for at least a minute. Find the minimum memory limit required.

- Create the pods defined in `design-review2.yaml`.

    ```bash
    kubectl create -f design-review2.yaml
    ```

    View all pods and their labels.

    ```bash
    kubectl get pods --show-labels
    ```

- Delete only the pods with the label `review=tux` using `--selector`. This should delete two of the four pods.

    ```bash
    kubectl delete pod --selector review=tux
    ```

- Create a new CronJob that runs `busybox` with the `sleep 30` command every 3 minutes. Verify it runs correctly. Then update the schedule so it runs at a specific time 10 minutes from now, every week (e.g. if it's 14:14, set it to run at `24 14 * * 1`).

- Delete all objects created during this review.

    ```bash
    kubectl delete pod design-pod1 label-pod3 label-pod4 --ignore-not-found
    kubectl delete cronjobs.batch sleepy --ignore-not-found
    kubectl delete deploy stressmeout --ignore-not-found
    ```
