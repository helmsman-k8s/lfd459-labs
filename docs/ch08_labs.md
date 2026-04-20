# Chapter 8 - Application Troubleshooting

## Lab Overview

This final chapter covers the troubleshooting flow for Kubernetes applications - from pod inspection to service endpoints to kube-proxy. You will work with API deprecations, use `kubectl debug` with ephemeral containers, and tackle a multi-bug deployment as a domain review.

---

## Exercise 8.1: Monitor Applications

A systematic troubleshooting flow is more valuable than memorising specific errors. Start from the application, work outward to the cluster.

1. Connect to the **controller** node. Check the `secondapp` pod status.

    ```bash
kubectl get pods secondapp
    NAME        READY   STATUS    RESTARTS   AGE
    secondapp   2/2     Running   49         2d
    ```

    The pod is running. `busy` has many restarts - expected, as `sleep 3600` completes every hour. `webserver` (nginx) should have 0 restarts.

2. Describe the pod in detail.

    ```bash
kubectl describe pod secondapp
    ```

    Check:

    - Both containers show `State: Running`
    - `webserver` has `Restart Count: 0`
    - `Conditions` all show `True`: `Initialized`, `Ready`, `PodScheduled`

3. Check pod `Conditions` specifically.

    ```bash
kubectl describe pod secondapp | grep -A8 "Conditions:"
    Conditions:
      Type              Status
      Initialized       True
      Ready             True
      PodScheduled      True
    ```

4. Scan the Events section for any `Warning` entries.

    ```bash
kubectl describe pod secondapp | grep -A20 "^Events:"
    ```

5. View the logs for each container.

    ```bash
kubectl logs secondapp webserver
kubectl logs secondapp busy
    ```

6. Exec into `busy` and test DNS and external connectivity.

    ```bash
kubectl exec -it secondapp -c busy -- sh
    ```

    Inside:

    ```
    / $ nslookup www.linuxfoundation.org
    Server:     10.96.0.10
    Name:       www.linuxfoundation.org
    Address: 23.185.0.2

    / $ cat /etc/resolv.conf
    nameserver 10.96.0.10
    search default.svc.cluster.local svc.cluster.local cluster.local

    / $ exit
    ```

7. Verify services and their selectors.

    ```bash
kubectl get svc
kubectl get svc secondapp -o yaml | grep -A5 "selector:"
    ```

    Confirm `selector.example: second` matches the pod label `example=second`.

8. Verify the endpoint object exists and has the correct pod IP and port.

    ```bash
kubectl get ep
    NAME         ENDPOINTS
    secondapp    192.168.x.y:80
    ...
kubectl get ep secondapp -o yaml
    ```

9. Check kube-proxy logs.

    ```bash
kubectl -n kube-system get pod | grep proxy
kubectl -n kube-system logs kube-proxy-<TAB>
    ```

10. Verify kube-proxy has created iptables rules for the service.

    ```bash
sudo iptables-save | grep secondapp
    ```

    You should see rules referencing port `32000` (NodePort) and the service ClusterIP.

11. Test via localhost NodePort to confirm the full proxy chain is working.

    ```bash
curl localhost:32000
    ```

---

## Exercise 8.2: Update a Deprecated YAML File

The Kubernetes API evolves. YAML that worked on older clusters may fail on newer ones due to API removals.

1. Move into the chapter lab directory and try to create the broken deployment.

    ```bash
    cd ~/lfd459/ch08-troubleshooting
    kubectl create -f brokendeploy.yaml
    ```

    The command will fail immediately. Read the error carefully.

2. Create a working deployment from scratch to understand what the current API requires.

    ```bash
kubectl create deployment reference --image=nginx --dry-run=client -o yaml
    ```

    Compare the output with `brokendeploy.yaml`.

3. Identify and fix all issues in `brokendeploy.yaml`. There are at least three:

    !!! note "Issues to find"
        - The `apiVersion` field references an API group that no longer exists
        - The `selector.matchLabels` field is missing entirely (required for `apps/v1` Deployments)
        - The pod template labels do not match the deployment's own labels

4. Edit the file and apply fixes.

    ```bash
vim brokendeploy.yaml
    ```

5. Create the deployment and verify the pod starts successfully.

    ```bash
kubectl create -f brokendeploy.yaml
kubectl get deployment broken
kubectl get pods -l app=broken
    ```

6. Clean up.

    ```bash
kubectl delete deployment broken
    ```

??? success "Fixes revealed"
    1. `apiVersion: extensions/v1beta1` -> `apiVersion: apps/v1` (removed in Kubernetes 1.16)
    2. Add `selector.matchLabels: {app: broken}` under `spec:` - mandatory for `apps/v1`
    3. Change pod template label from `app: thirdpage` to `app: broken` so the selector matches

---

## Exercise 8.3: Troubleshooting with Ephemeral Containers

Ephemeral containers allow you to attach a debugging shell to a running pod - even if the original container image has no shell.

1. Deploy the broken application.

    ```bash
kubectl apply -f brokenapp.yaml
    pod/nginx-debug-pod created
    service/nginx-debug-svc created
    ```

2. Check the pod status. It will show `0/1 READY`.

    ```bash
kubectl get all
    NAME                   READY   STATUS    RESTARTS   AGE
    pod/nginx-debug-pod    0/1     Running   0          5s

    NAME                     TYPE        CLUSTER-IP       PORT(S)   AGE
    service/nginx-debug-svc  ClusterIP   10.111.195.30    80/TCP    5s
    ```

3. Try to connect to the service - it will fail as the pod has no endpoints yet.

    ```bash
kubectl get ep nginx-debug-svc
    NAME              ENDPOINTS   AGE
    nginx-debug-svc   <none>      10s
curl 10.111.195.30
    curl: (7) Failed to connect to 10.111.195.30 port 80
    ```

4. Review the pod's probes.

    ```bash
cat brokenapp.yaml
    ```

    Two things are happening:

    - The **startupProbe** runs `rm -f /bin/bash` - it deletes the bash shell from the container
    - The **readinessProbe** checks for `/tmp/app.log` - this file does not exist yet

5. Try to exec directly into the pod - it will fail because `/bin/bash` has been removed.

    ```bash
kubectl exec -it pod/nginx-debug-pod -- bash
    error: Internal error occurred: exec: "bash": executable file not found in $PATH
    ```

6. Attach an **ephemeral container** using `kubectl debug`.

    ```bash
kubectl debug pod/nginx-debug-pod -it \
    --image=busybox --target=nginx -- /bin/sh
    ```

7. Inside the debug container, verify nginx is running as PID 1.

    ```
    # ps aux
    PID   USER   COMMAND
    1     root   nginx: master process nginx -g daemon off;
    29    root   nginx: worker process
    30    root   sh
    ```

8. Confirm you are inside the same pod namespace.

    ```
    # hostname
    nginx-debug-pod
    ```

9. Create the missing `/tmp/app.log` file by writing through `/proc/1/root`.

    ```
    # echo "hello from debug" > /proc/1/root/tmp/app.log
    # ls -l /proc/1/root/tmp/
    -rw-r--r-- 1 root root 17 ... app.log
    # cat /proc/1/root/tmp/app.log
    hello from debug
    # exit
    ```

10. The readiness probe succeeds within the next `periodSeconds`. Verify the pod is now Ready.

    ```bash
kubectl get all
    NAME                   READY   STATUS    RESTARTS   AGE
    pod/nginx-debug-pod    1/1     Running   0          2m

    NAME                     TYPE        CLUSTER-IP      ENDPOINTS
    service/nginx-debug-svc  ClusterIP   10.111.195.30   10.244.x.y:80
    ```

11. Test service access.

    ```bash
curl 10.111.195.30
    <!DOCTYPE html>
    <html><head><title>Welcome to nginx!</title></head>...
    ```

12. Clean up.

    ```bash
kubectl delete -f brokenapp.yaml
    ```

!!! note "Key takeaway"
    `kubectl debug` with ephemeral containers is the correct tool when the container has no shell, the image is distroless, or when you need to debug without modifying the running container. The `--target` flag makes the ephemeral container share the target container's process namespace.

---

## Exercise 8.4: Conformance Testing (Optional)

!!! note "Optional"
    Sonobuoy conformance testing takes 60+ minutes to complete and is not required for the CKAD exam.

```bash
# Download the latest release
curl -sLO \
https://github.com/vmware-tanzu/sonobuoy/releases/latest/download/sonobuoy_linux_amd64.tar.gz
tar -xvf sonobuoy_linux_amd64.tar.gz
sudo mv sonobuoy /usr/local/bin/

# Run tests
sonobuoy run --wait

# Monitor progress in a second terminal
sonobuoy status
sonobuoy logs

# Retrieve and view results
sonobuoy retrieve
sonobuoy results <tarball-file>

# Clean up
sonobuoy delete --wait
```

---

## Exercise 8.5: Domain Review

!!! important
    Source pages and content in this review can change at any time. Always check current information before your exam.

Revisit the CKAD curriculum for topics covered in this chapter:

- Understand API deprecations
- Use provided tools to monitor Kubernetes applications
- Debugging in Kubernetes

- Use `troubleshoot-review1.yaml` to create a deployment. The create command will fail immediately.

    ```bash
kubectl create -f troubleshoot-review1.yaml
    ```

- Read the error carefully. Then inspect the file and identify **all** the problems. There are at least four distinct issues.

    ```bash
cat troubleshoot-review1.yaml
    ```

    !!! note "Where to look"
        Check the `apiVersion`, `replicas`, `selector.matchLabels` vs pod template labels, and the resource `requests` vs `limits`.

- Fix each issue and recreate until the deployment runs a single pod for at least one minute without errors.

    ```bash
kubectl get deploy igottrouble
    NAME          READY   UP-TO-DATE   AVAILABLE   AGE
    igottrouble   1/1     1            1           5m13s
    ```

- Clean up.

    ```bash
kubectl delete deploy igottrouble --ignore-not-found
    ```

??? success "Fixes revealed"
    1. `apiVersion: extensions/v1` -> `apiVersion: apps/v1` (invalid group)
    2. `replicas: 0` -> `replicas: 1`
    3. `selector.matchLabels: run: ugottrouble` -> `run: igottrouble` (typo)
    4. CPU `requests: 2.5` exceeds `limits: 1` - fix: `requests.cpu: "0.5"`

---

## Course Complete

Congratulations on completing all LFD459 lab chapters. Before sitting the CKAD exam, make sure you:

- Can create, edit, and troubleshoot all object types covered without referencing notes
- Are familiar with the three documentation sources allowed during the exam
- Have practised the domain review exercises at speed
- Have reviewed the current CKAD curriculum at <https://www.cncf.io/certification/ckad/>

Good luck!
