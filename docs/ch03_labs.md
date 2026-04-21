# Chapter 3 - Build

## Lab Overview

In this chapter you will containerize a simple Python application using Podman, set up a local container registry inside the cluster, push your image to it, deploy the application with multiple replicas, and configure readiness and liveness probes to manage container health.

---

## Exercise 3.1: Deploy a New Application

### Working with a Simple Python Script

1. Connect to the **controller** node. Verify Python 3 is installed.

    ```bash
    sudo apt-get -y install python3
    ```
    # output

    It should report already installed on Ubuntu 22.04.

2. Move into the chapter lab directory.

    ```bash
    cd ~/lfd459/ch03-build
    ls
    ```

    You should see: `simple.py`, `Dockerfile`, `easyregistry.yaml`, `local-repo-setup.sh`, `edited-simpleapp.yaml`, `edited-later-simpleapp.yaml`, `build-review1.yaml`

3. Create a working directory for the application and move into it.

    ```bash
    mkdir app1
    cd app1
    ```
    # output

4. Copy `simple.py` into the app1 directory and review it.

    ```bash
    cp ../simple.py .
    cat simple.py
    ```

    The script runs in an infinite loop, writing the hostname and current timestamp to a file called `date.out` every 5 seconds.

5. Make the script executable and run it briefly to verify it works. Use **Ctrl-C** to stop it after 15-20 seconds.

    ```bash
    chmod +x simple.py
    ```
    # output

    ```
    ./simple.py
    ^CTraceback (most recent call last):
      File "./simple.py", line 24, in <module>
        time.sleep(5)
    KeyboardInterrupt
    ```
    # output

6. Verify the output file was created with hostname and timestamp entries.

    ```bash
    cat date.out
    ```

    ```
    # output
    2026-03-18 19:27:50
    controller
    2026-03-18 19:27:55
    controller
    ...
    ```

7. Copy the `Dockerfile` into the app1 directory and review it.

    ```bash
    cp ../Dockerfile .
    cat Dockerfile
    ```
    # output

    ```dockerfile
    FROM docker.io/library/python:3
    ADD simple.py /
    ## RUN pip install pystrich
    CMD [ "python", "./simple.py" ]
    ```

    !!! note
        The filename must be exactly `Dockerfile` - capital D, no extension.

8. Install Podman. Podman is a daemonless container tool compatible with Docker syntax.

    ```bash
    sudo apt-get install -y podman
    ```
    # output

9. Build the container image. The dot (`.`) at the end means "use the current directory as the build context". When prompted to choose a registry for the `python` base image, select `docker.io`.

    ```bash
    sudo podman build -t simpleapp .
    ```

    ```
    # output
    STEP 1/3: FROM docker.io/library/python:3
    Trying to pull docker.io/library/python:3...
    ...
    Successfully tagged localhost/simpleapp:latest
    ```

10. Verify the new image appears in the local image list.

    ```bash
    sudo podman images
    ```
    # output

    ```
    REPOSITORY              TAG     IMAGE ID      CREATED        SIZE
    localhost/simpleapp     latest  11d4607c72e0  7 seconds ago  1.04 GB
    docker.io/library/python  3     58a8f3dcd68a  3 days ago     1.04 GB
    ```
    # output

11. Run the container image with Podman. The script will run silently in the foreground. Wait 15 seconds then use **Ctrl-C** to stop it.

    ```bash
    sudo podman run localhost/simpleapp
    ```

    ```
    # output
    ^CTraceback (most recent call last):
      File "./simple.py", line 24, in <module>
        time.sleep(5)
    KeyboardInterrupt
    ```

12. Locate the `date.out` file that Podman created inside the container's overlay filesystem. Note the container hostname is a random hash, not `controller`.

    ```bash
    sudo find / -name date.out
    ```
    # output

    ```
    /home/guru/app1/date.out
    /var/lib/containers/storage/overlay/.../diff/date.out
    ```
    # output

13. View the contents of the container's `date.out` file. Use the full path returned by the find command above.

    ```bash
    sudo tail /var/lib/containers/storage/overlay/<hash>/diff/date.out
    ```

    ```
    # output
    2026-03-18 19:32:20
    b8cec851efca
    2026-03-18 19:32:25
    b8cec851efca
    ```

---

## Exercise 3.2: Configure a Local Registry

Rather than pushing to Docker Hub, we will deploy a private registry inside the cluster and make it available to all nodes.

1. Return to the chapter directory and create the local registry using the provided YAML.

    ```bash
    cd ~/lfd459/ch03-build
    kubectl create -f easyregistry.yaml
    ```
    # output

    ```
    service/nginx created
    service/registry created
    deployment.apps/nginx created
    persistentvolumeclaim/nginx-claim0 created
    deployment.apps/registry created
    persistentvolumeclaim/registry-claim0 created
    persistentvolume/vol1 created
    persistentvolume/vol2 created
    ```
    # output

2. Note the ClusterIP assigned to the registry service. In this setup it is expected to be `10.97.40.62` - verify this matches.

    ```bash
    kubectl get svc | grep registry
    ```

    ```
    # output
    registry   ClusterIP   10.97.40.62   <none>   5000/TCP   5m35s
    ```

    !!! warning "If the ClusterIP is different"
        The `easyregistry.yaml` hardcodes `clusterIP: 10.97.40.62`. If a different IP was assigned, delete the service, edit the YAML to match the actual IP, and recreate it. The `local-repo-setup.sh` script also uses this same IP - edit it before running if needed.

3. Verify the registry is responding.

    ```bash
    curl 10.97.40.62:5000/v2/_catalog
    ```
    # output

    ```
    {"repositories":[]}
    ```
    # output

4. Configure containerd and Podman on the **controller** to use the local registry over HTTP. Run the setup script (note the dot-space to source it and export the `$repo` variable into your shell).

    ```bash
    chmod +x local-repo-setup.sh
    . ./local-repo-setup.sh
    ```

    ```
    # output
    Configuring local repo, Please standby
    ...
    Local Repo configured, follow the next steps
    ```

5. Pull a small test image from Docker Hub, tag it for the local registry, and push it.

    ```bash
    sudo podman pull docker.io/library/alpine
    sudo podman tag alpine $repo/tagtest
    sudo podman push $repo/tagtest
    ```
    # output

    ```
    Getting image source signatures
    Copying blob b2d5eeeaba3a done
    ...
    ```
    # output

6. Remove the local cached copies of the images, then pull from your local registry to verify it works end-to-end.

    ```bash
    sudo podman image rm alpine
    sudo podman image rm $repo/tagtest
    sudo podman pull $repo/tagtest
    ```

    ```
    # output
    Trying to pull 10.97.40.62:5000/tagtest:latest...
    ...
    6dbb9cc54074106d46d4ccb330f2a40a682d49dda5f4844962b7dce9fe44aaec
    ```

7. Connect to **worker1** in a separate terminal. Install Podman and configure the local registry on the worker node the same way.

    ```bash
    sudo apt-get install -y podman
    scp guru@controller:~/lfd459/ch03-build/local-repo-setup.sh ~/
    chmod +x ~/local-repo-setup.sh
    . ~/local-repo-setup.sh
    ```
    # output

    Verify the worker can pull from the registry.

    ```bash
    sudo podman pull $repo/tagtest
    ```

    ```
    # output
    Trying to pull 10.97.40.62:5000/tagtest:latest...
    ...
    ```

    Repeat on **worker2**.

8. Return to the **controller**. Tag and push the `simpleapp` image to the local registry.

    ```bash
    cd app1
    sudo podman tag simpleapp $repo/simpleapp
    sudo podman push $repo/simpleapp
    ```
    # output

    ```
    Getting image source signatures
    Copying blob 47458fb45d99 done
    ...
    ```
    # output

9. Verify both images appear in the registry catalog from the controller.

    ```bash
    curl $repo/v2/_catalog
    ```

    ```
    # output
    {"repositories":["simpleapp","tagtest"]}
    ```

10. Deploy the simpleapp from the local registry and scale it to 6 replicas.

    ```bash
    kubectl create deployment try1 --image=$repo/simpleapp
    ```
    # output

    ```
    deployment.apps/try1 created
    ```
    # output

    ```bash
    kubectl scale deployment try1 --replicas=6
    ```

    ```
    # output
    deployment.apps/try1 scaled
    ```

    ```bash
    kubectl get pod -o wide
    ```
    # output

    ```
    NAME                   READY  STATUS   RESTARTS  AGE  IP              NODE
    try1-55f675ddd-28vgs   1/1    Running  0         17s  10.244.1.30     worker1
    try1-55f675ddd-4vzt7   1/1    Running  0         17s  10.244.2.100    worker2
    try1-55f675ddd-2nrsj   1/1    Running  0         17s  10.244.0.26     controller
    ...
    ```
    # output

    Pods should spread across all three nodes.

11. On **worker1**, verify the simpleapp containers are running via crictl.

    ```bash
    sudo crictl config \
    --set runtime-endpoint=unix:///run/containerd/containerd.sock \
    --set image-endpoint=unix:///run/containerd/containerd.sock
    ```

    ```bash
    sudo crictl ps | grep simple
    ```
    # output

12. Return to the **controller**. Save the try1 deployment as a YAML file.

    ```bash
    kubectl get deployment try1 -o yaml > simpleapp.yaml
    ```

13. Delete and recreate the deployment from the saved YAML. Verify 6/6 replicas are running.

    ```bash
    kubectl delete deployment try1
    ```
    # output

    ```
    deployment.apps "try1" deleted
    ```
    # output

    ```bash
    kubectl create -f simpleapp.yaml
    ```

    ```
    # output
    deployment.apps/try1 created
    ```

    ```bash
    kubectl get deployment
    ```
    # output

    ```
    NAME       READY   UP-TO-DATE   AVAILABLE   AGE
    nginx      1/1     1            1           15m
    registry   1/1     1            1           15m
    try1       6/6     6            6           5s
    ```
    # output

---

## Exercise 3.3: Configure Probes

### readinessProbe

A `readinessProbe` tells Kubernetes when a container is ready to accept traffic. Until the probe succeeds, the pod is not sent any requests.

1. Edit `simpleapp.yaml` and add a `readinessProbe` to the `simpleapp` container. The probe runs `cat /tmp/healthy` - the container is not considered ready until that file exists.

    ```bash
    vim simpleapp.yaml
    ```

    Find the `containers:` section for the simpleapp container and add the `readinessProbe` block after the `imagePullPolicy` line:

    ```yaml
    spec:
      containers:
      - image: 10.97.40.62:5000/simpleapp
        imagePullPolicy: Always
        name: simpleapp
        readinessProbe:
          periodSeconds: 5
          exec:
            command:
            - cat
            - /tmp/healthy
        resources: {}
    ```
    # output

    !!! tip
        The `edited-simpleapp.yaml` file in your working directory already has this probe added. You can reference it for the correct indentation, but remember to verify the registry IP matches your environment.

2. Delete and recreate the deployment.

    ```bash
    kubectl delete deployment try1
    ```

    ```
    # output
    deployment.apps "try1" deleted
    ```

    ```bash
    kubectl create -f simpleapp.yaml
    ```
    # output

    ```
    deployment.apps/try1 created
    ```
    # output

3. The deployment shows 6 pods but **0 available** - all are waiting for `/tmp/healthy` to exist.

    ```bash
    kubectl get deployment
    ```

    ```
    # output
    NAME       READY   UP-TO-DATE   AVAILABLE   AGE
    try1       0/6     6            0           15s
    ```

4. Check the pods. Each try1 pod shows `0/1` in READY - the readinessProbe is failing.

    ```bash
    kubectl get pods
    ```
    # output

    ```
    NAME                    READY   STATUS    RESTARTS   AGE
    try1-9869bdb88-rtchc    0/1     Running   0          26s
    try1-9869bdb88-2wfnr    0/1     Running   0          26s
    ...
    ```
    # output

5. Exec into one pod and create the health check file.

    ```bash
    kubectl exec -it try1-9869bdb88-rtchc -- /bin/bash
    ```

    ```
    # output
    root@try1-9869bdb88-rtchc:/# touch /tmp/healthy
    root@try1-9869bdb88-rtchc:/# exit
    ```

6. Wait at least 5 seconds (the `periodSeconds`), then check pods again. The pod you touched should now show `1/1`.

    ```bash
    kubectl get pods
    ```
    # output

    ```
    NAME                    READY   STATUS    RESTARTS   AGE
    try1-9869bdb88-rtchc    1/1     Running   0          4m
    try1-9869bdb88-2wfnr    0/1     Running   0          4m
    ...
    ```
    # output

7. Use a `for` loop to touch `/tmp/healthy` in all remaining pods at once.

    ```bash
    for name in $(kubectl get pods -l app=try1 -o name); \
    do kubectl exec $name -- touch /tmp/healthy; done
    ```

8. Wait a few seconds, then verify all 6 pods are now `1/1 Ready`.

    ```bash
    kubectl get pods
    ```
    # output

    ```
    NAME                    READY   STATUS    RESTARTS   AGE
    try1-9869bdb88-rtchc    1/1     Running   0          22m
    try1-9869bdb88-2wfnr    1/1     Running   0          22m
    ...
    ```
    # output

### livenessProbe and Sidecar Container

1. Edit `simpleapp.yaml` again to add a `livenessProbe` and a second sidecar container (`goproxy`). Find the end of the simpleapp container definition and add the goproxy container after it.

    ```bash
    vim simpleapp.yaml
    ```

    Add after `terminationMessagePolicy: File`:

    ```yaml
        - name: goproxy
          image: registry.k8s.io/goproxy:0.1
          ports:
          - containerPort: 8080
          readinessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
    ```
    # output

    !!! tip
        The `edited-later-simpleapp.yaml` file shows the complete deployment with both containers configured. Use it as a reference for indentation.

2. Delete and recreate the deployment.

    ```bash
    kubectl delete deployment try1
    ```

    ```
    # output
    deployment.apps "try1" deleted
    ```

    ```bash
    kubectl create -f simpleapp.yaml
    ```
    # output

    ```
    deployment.apps/try1 created
    ```
    # output

3. Check the pods. Each pod now has two containers (`READY 1/2` or `0/2`). The goproxy container starts, but simpleapp is not ready again because `/tmp/healthy` is missing.

    ```bash
    kubectl get pods
    ```

    ```
    # output
    NAME                      READY   STATUS    RESTARTS   AGE
    try1-76cc5ffcc6-4rjvh     1/2     Running   0          3s
    try1-76cc5ffcc6-bk5f5     1/2     Running   0          3s
    try1-76cc5ffcc6-d8n5q     0/2     Running   0          3s
    ...
    ```

4. Create the health check file in all pods. Specify the `-c simpleapp` flag to target the correct container.

    ```bash
    for name in $(kubectl get pods -l app=try1 -o name); \
    do kubectl exec $name -c simpleapp -- touch /tmp/healthy; done
    ```
    # output

5. Wait a minute and verify all pods show `2/2 Running`.

    ```bash
    kubectl get pods
    ```

    ```
    # output
    NAME                      READY   STATUS    RESTARTS   AGE
    try1-76cc5ffcc6-4rjvh     2/2     Running   0          ~1m
    try1-76cc5ffcc6-bk5f5     2/2     Running   0          ~1m
    ...
    ```

6. Examine the events for one of the pods. Note the `Readiness probe failed` warnings from before the file was created.

    ```bash
    kubectl describe pod try1-76cc5ffcc6-4rjvh | tail -20
    ```
    # output

    Look for `Warning Unhealthy` events showing `cat: /tmp/healthy: No such file or directory`.

7. Confirm both containers in the pod show `State: Running` and `Ready: True`.

    ```bash
    kubectl describe pod try1-76cc5ffcc6-4rjvh | grep -E 'State|Ready'
    ```

    ```
    # output
    State:          Running
    Ready:          True
    State:          Running
    Ready:          True
    Ready:          True
    ContainersReady: True
    ```

    !!! warning "Troubleshooting"
        If the try1 deployment seems stuck, check the registry catalog and re-push if needed, then scale down to 0 and back to 6:

        ```bash
        curl $repo/v2/_catalog
        sudo podman push $repo/simpleapp
        kubectl scale deployment try1 --replicas 0
        kubectl scale deployment try1 --replicas 6
        ```

---

## Exercise 3.4: Domain Review

!!! important
    Source pages and content in this review can change at any time. Always check current information.

Revisit the CKAD curriculum and locate the topics covered in this chapter:

- Implement probes and health checks
- Define, build and modify container images
- Understand multi-container Pod design patterns
- Utilize container logs

- Using the three documentation sources allowed during the CKAD exam, find and bookmark working YAML examples for `livenessProbe`, `readinessProbe`, and multi-container pods.
- Deploy a new nginx webserver. Add both a `livenessProbe` and a `readinessProbe` on port 80. Test that both probes and the webserver are working.
- Use `build-review1.yaml` to create a broken deployment. Fix it so both containers are running and in a `READY` state.

    ```bash
        ```

        ```bash
    kubectl create -f build-review1.yaml
        ```

    !!! hint
        The web server (`nginx`) listens on port 80. The proxy (`goproxy`) listens on port 8080. Examine the probe configurations carefully - one of the port numbers is wrong.

- Once fixed, access the default nginx page and verify the GET request appears in the container log.

    Get the pod IP:

    ```bash
    kubectl get pod -l app=break2 -o wide
    ```
    # output

    Then curl the pod IP and check the log:

    ```bash
    curl http://<pod-ip from above>
    kubectl logs <pod-name> -c brokenapp
    ```

    You should see a line like:

    ```
    # output
    192.168.124.0 - - [21/Mar/2026:03:30:31 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.58.0"
    ```

- Clean up all resources created in this review before moving on.

    ```bash
    kubectl delete deployment try1 nginx registry break2 --ignore-not-found
    kubectl delete svc nginx registry --ignore-not-found
    kubectl delete pvc nginx-claim0 registry-claim0 --ignore-not-found
    kubectl delete pv vol1 vol2 --ignore-not-found
    ```
    # output
