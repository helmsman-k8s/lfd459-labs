# Chapter 8 ? Application Troubleshooting

## Lab Overview

This final chapter covers the troubleshooting flow for Kubernetes applications ? from pod inspection to service endpoints to kube-proxy. You will work with API deprecations, use `kubectl debug` with ephemeral containers, and tackle a multi-bug deployment as a domain review.

---

## Exercise 8.1: Monitor Applications

A systematic troubleshooting flow is more valuable than memorising specific errors. Start from the application, work outward to the cluster.

1. Connect to the **controller** node. Check the `secondapp` pod status.

    ```bash
    guru@controller:~$ kubectl get pods secondapp
    NAME        READY   STATUS    RESTARTS   AGE
    secondapp   2/2     Running   49         2d
    ```

    The pod is running. Note `busy` has many restarts ? this is expected, as the `sleep 3600` command completes every hour and the container is restarted. `webserver` (nginx) should have 0 restarts.

2. Describe the pod in detail. Work through every section.

    ```bash
    guru@controller:~$ kubectl describe pod secondapp
    ```

    Check:
    - Both containers (`webserver` and `busy`) show `State: Running`
    - `webserver` has `Restart Count: 0`
    - `busy` has a high restart count matching its runtime in hours
    - `Conditions` all show `True`: `Initialized`, `Ready`, `PodScheduled`

3. Check pod `Conditions` specifically.

    ```bash
    guru@controller:~$ kubectl describe pod secondapp | grep -A8 "Conditions:"
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      PodScheduled      True
    ```

4. Scan the Events section at the bottom of `describe` output for any `Warning` entries. Repeated `Pulling` events for `busybox` are normal ? they accompany each restart.

    ```bash
    guru@controller:~$ kubectl describe pod secondapp | grep -A20 "^Events:"
    ```

5. View the logs for each container. `webserver` should show HTTP access log entries. `busy` produces no output.

    ```bash
    guru@controller:~$ kubectl logs secondapp webserver
    guru@controller:~$ kubectl logs secondapp busy
    ```

6. Exec into `busy` and test DNS and external connectivity.

    ```bash
    guru@controller:~$ kubectl exec -it secondapp -c busy -- sh
    ```

    Inside:

    ```sh
    / $ nslookup www.linuxfoundation.org
    Server:     10.96.0.10
    Name:       www.linuxfoundation.org
    Address: 23.185.0.2

    / $ cat /etc/resolv.conf
    nameserver 10.96.0.10
    search default.svc.cluster.local svc.cluster.local cluster.local

    / $ nc -vz www.linux.com 25
    # If this hangs, Ctrl-C ? port 25 (SMTP) may be blocked by the lab network

    / $ wget http://www.linux.com/
    # May get "Permission denied" writing index.html ? this is expected
    # with runAsUser: 2000. Connectivity itself works.

    / $ exit
    ```

7. Verify services and their selectors. A selector typo is a common cause of traffic not reaching pods.

    ```bash
    guru@controller:~$ kubectl get svc
    guru@controller:~$ kubectl get svc secondapp -o yaml | grep -A5 "selector:"
    ```

    Confirm `selector.example: second` matches the pod label `example=second`.

8. Verify the endpoint object exists and has the correct pod IP and port. An empty `ENDPOINTS` field means the selector matched nothing ? no pods are ready.

    ```bash
    guru@controller:~$ kubectl get ep
    NAME         ENDPOINTS
    secondapp    192.168.x.y:80
    ...

    guru@controller:~$ kubectl get ep secondapp -o yaml
    ```

9. If a service is not routing traffic, check kube-proxy. On kubeadm clusters it runs as a pod in `kube-system`.

    ```bash
    guru@controller:~$ kubectl -n kube-system get pod | grep proxy
    guru@controller:~$ kubectl -n kube-system logs kube-proxy-<Tab>
    ```

    Lines beginning with `I` are informational, `E` are errors. Any RBAC-related errors would indicate a permissions problem.

10. Verify kube-proxy has created iptables rules for the service.

    ```bash
    guru@controller:~$ sudo iptables-save | grep secondapp
    ```

    You should see rules referencing port `32000` (NodePort) and the service ClusterIP.

11. Test via localhost NodePort to confirm the proxy chain is working end-to-end.

    ```bash
    guru@controller:~$ curl localhost:32000
    ```

    A successful nginx response confirms the full path ? service ? endpoint ? pod ? is working.

---

## Exercise 8.2: Update a Deprecated YAML File

The Kubernetes API evolves. YAML that worked on older clusters may fail on newer ones due to API removals.

1. Copy the chapter lab files and try to create the broken deployment.

    ```bash
    guru@controller:~$ cp ~/lfd459/ch08-troubleshooting/* .
    guru@controller:~$ kubectl create -f brokendeploy.yaml
    ```

    The command will fail immediately. Read the error carefully.

2. Create a working deployment from scratch to understand what the current API requires.

    ```bash
    guru@controller:~$ kubectl create deployment reference --image=nginx --dry-run=client -o yaml
    ```

    Compare the output with `brokendeploy.yaml`.

3. Identify and fix all issues in `brokendeploy.yaml`. There are at least three:

    !!! hint "Issues to find"
        - The `apiVersion` field references an API group that no longer exists
        - The `selector.matchLabels` field is missing entirely (required for `apps/v1` Deployments)
        - The pod template labels do not match the deployment's own labels

4. Edit the file and apply fixes.

    ```bash
    guru@controller:~$ vim brokendeploy.yaml
    ```

5. Create the deployment and verify the pod starts successfully.

    ```bash
    guru@controller:~$ kubectl create -f brokendeploy.yaml
    guru@controller:~$ kubectl get deployment broken
    guru@controller:~$ kubectl get pods -l app=broken
    ```

6. Clean up.

    ```bash
    guru@controller:~$ kubectl delete deployment broken
    ```

??? success "Fixes revealed"
    1. `apiVersion: extensions/v1beta1` ? `apiVersion: apps/v1` (`extensions/v1beta1` was removed in Kubernetes 1.16)
    2. Add `selector.matchLabels: {app: broken}` under `spec:` ? mandatory for `apps/v1`
    3. Change pod template label from `app: thirdpage` to `app: broken` so the selector matches

---

## Exercise 8.3: Troubleshooting with Ephemeral Containers

Ephemeral containers allow you to attach a debugging shell to a running pod ? even if the original container image has no shell. This is the modern replacement for exec when the shell has been removed or was never present.

1. Deploy the broken application.

    ```bash
    guru@controller:~$ kubectl apply -f brokenapp.yaml
    pod/nginx-debug-pod created
    service/nginx-debug-svc created
    ```

2. Check the pod status. It will show `0/1 READY` ? running but not ready.

    ```bash
    guru@controller:~$ kubectl get all
    NAME                   READY   STATUS    RESTARTS   AGE
    pod/nginx-debug-pod    0/1     Running   0          5s

    NAME                   TYPE        CLUSTER-IP       PORT(S)   AGE
    service/nginx-debug-svc ClusterIP  10.111.195.30    80/TCP    5s
    ```

3. Try to connect to the service. It will fail ? the pod is not ready, so it has no endpoints.

    ```bash
    guru@controller:~$ kubectl get ep nginx-debug-svc
    NAME              ENDPOINTS   AGE
    nginx-debug-svc   <none>      10s

    guru@controller:~$ curl 10.111.195.30
    curl: (7) Failed to connect to 10.111.195.30 port 80
    ```

4. Describe the service to confirm the problem is the missing endpoint (not a selector mismatch).

    ```bash
    guru@controller:~$ kubectl describe svc nginx-debug-svc
    ```

    The selector matches, but `Endpoints: <none>` confirms the pod is not yet passing its readiness probe.

5. Review the pod's probes in `brokenapp.yaml`.

    ```bash
    guru@controller:~$ cat brokenapp.yaml
    ```

    Two things are happening:
    - The **startupProbe** runs `rm -f /bin/bash` ? it deletes the bash shell from the container
    - The **readinessProbe** checks for `/tmp/app.log` ? this file doesn't exist yet

6. Try to exec directly into the pod ? it will fail because `/bin/bash` has been removed.

    ```bash
    guru@controller:~$ kubectl exec -it pod/nginx-debug-pod -- bash
    error: Internal error occurred: exec: "bash": executable file not found in $PATH
    ```

7. Attach an **ephemeral container** using `kubectl debug`. This injects a busybox shell into the pod without modifying the original container.

    ```bash
    guru@controller:~$ kubectl debug pod/nginx-debug-pod -it \
    --image=busybox --target=nginx -- /bin/sh
    ```

    Inside the debug container:

8. Verify nginx is running as PID 1 in the target container.

    ```sh
    # ps aux
    PID   USER   COMMAND
    1     root   nginx: master process nginx -g daemon off;
    29    root   nginx: worker process
    30    root   sh
    ```

9. Confirm you are inside the same pod namespace.

    ```sh
    # hostname
    nginx-debug-pod
    ```

10. Create the missing `/tmp/app.log` file by writing through `/proc/1/root` ? the root filesystem of the target container (PID 1 = nginx).

    ```sh
    # echo "hello from debug" > /proc/1/root/tmp/app.log
    # ls -l /proc/1/root/tmp/
    -rw-r--r-- 1 root root 17 ... app.log
    # cat /proc/1/root/tmp/app.log
    hello from debug
    # exit
    ```

11. After exiting, the readiness probe succeeds within the next `periodSeconds` (10 seconds). The pod becomes Ready and the service endpoint is populated.

    ```bash
    guru@controller:~$ kubectl get all
    NAME                   READY   STATUS    RESTARTS   AGE
    pod/nginx-debug-pod    1/1     Running   0          2m

    NAME                   TYPE        CLUSTER-IP      ENDPOINTS
    service/nginx-debug-svc ClusterIP  10.111.195.30   10.244.x.y:80
    ```

12. Test service access ? it now works.

    ```bash
    guru@controller:~$ curl 10.111.195.30
    <!DOCTYPE html>
    <html><head><title>Welcome to nginx!</title></head>...
    ```

13. Clean up.

    ```bash
    guru@controller:~$ kubectl delete -f brokenapp.yaml
    ```

!!! note "Key takeaway"
    `kubectl debug` with ephemeral containers is the correct tool when: the container has no shell, the image is distroless, or when you need to debug without modifying the running container. The `--target` flag makes the ephemeral container share the target container's process namespace.

---

## Exercise 8.4: Conformance Testing (Optional)

!!! note "Optional"
    Sonobuoy conformance testing takes 60+ minutes to complete and is not required for the CKAD exam. It is included here for those who want to verify their cluster's conformance after the course.

Sonobuoy by VMware Tanzu runs the Kubernetes e2e test suite against your cluster and reports conformance results.

```bash
# Download the latest release
guru@controller:~$ curl -sLO \
https://github.com/vmware-tanzu/sonobuoy/releases/latest/download/sonobuoy_linux_amd64.tar.gz
guru@controller:~$ tar -xvf sonobuoy_linux_amd64.tar.gz
guru@controller:~$ sudo mv sonobuoy /usr/local/bin/

# Run tests (--wait blocks until complete ? up to 60+ minutes)
guru@controller:~$ sonobuoy run --wait

# In a second terminal, monitor progress
guru@controller:~$ sonobuoy status
guru@controller:~$ sonobuoy logs

# When done, retrieve and view results
guru@controller:~$ sonobuoy retrieve
guru@controller:~$ sonobuoy results <tarball-file>

# Clean up
guru@controller:~$ sonobuoy delete --wait
```

---

## Exercise 8.5: Domain Review

!!! warning "Important"
    Source pages and content in this review can change at any time. Always check current information before your exam.

Revisit the CKAD curriculum for topics covered in this chapter:

- Understand API deprecations
- Use provided tools to monitor Kubernetes applications
- Debugging in Kubernetes

1. Use `troubleshoot-review1.yaml` to create a deployment. The create command will fail immediately.

    ```bash
    guru@controller:~$ kubectl create -f troubleshoot-review1.yaml
    ```

2. Read the error carefully. Then inspect the file and identify **all** the problems. There are at least four distinct issues.

    ```bash
    guru@controller:~$ cat troubleshoot-review1.yaml
    ```

    !!! hint "Where to look"
        Check the `apiVersion`, `replicas`, `selector.matchLabels` vs pod template labels, and the resource `requests` vs `limits`.

3. Fix each issue and recreate until the deployment runs a single pod for at least one minute without errors.

    ```bash
    guru@controller:~$ kubectl get deploy igottrouble
    NAME          READY   UP-TO-DATE   AVAILABLE   AGE
    igottrouble   1/1     1            1           5m13s
    ```

4. Clean up.

    ```bash
    guru@controller:~$ kubectl delete deploy igottrouble --ignore-not-found
    ```

??? success "Fixes revealed"
    1. `apiVersion: extensions/v1` ? `apiVersion: apps/v1` (invalid group)
    2. `replicas: 0` ? `replicas: 1`
    3. `selector.matchLabels: run: ugottrouble` ? `run: igottrouble` (typo ? doesn't match pod template label)
    4. CPU `requests: 2.5` exceeds `limits: 1` ? requests cannot exceed limits. Fix: `requests.cpu: "0.5"` (or raise the limit)

---

## Course Complete

Congratulations on completing all LFD459 lab chapters. Before sitting the CKAD exam, make sure you:

- Can create, edit, and troubleshoot all object types covered without referencing notes
- Are familiar with the three documentation sources allowed during the exam
- Have practised the domain review exercises at speed
- Have reviewed the current CKAD curriculum at [https://www.cncf.io/certification/ckad/](https://www.cncf.io/certification/ckad/)

Good luck!
