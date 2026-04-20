# Chapter 2 - Kubernetes Architecture

## Lab Overview

In this chapter you will explore the Kubernetes cluster that has been pre-installed for you, create your first pods and services, and understand how the control plane manages workloads through deployments and ReplicaSets.

---

## Exercise 2.1: Verify the Cluster

The cluster is already installed and ready. Your first task is to confirm everything is healthy before starting the labs.

1. Connect to the **controller** node from your browser.

2. Verify both worker nodes have joined the cluster and show a `Ready` state.

    ```bash
    guru@controller:~$ kubectl get nodes
    ```

    Expected output:

    ```
    NAME         STATUS   ROLES           AGE   VERSION
    controller   Ready    control-plane   Xm    v1.33.1
    worker1      Ready    worker          Xm    v1.33.1
    worker2      Ready    worker          Xm    v1.33.1
    ```

    !!! note "Worker node roles"
        By default, worker nodes show `<none>` for the ROLES column. If your output shows `<none>`, label them with:
        ```bash
        guru@controller:~$ kubectl label node worker1 node-role.kubernetes.io/worker=worker
        guru@controller:~$ kubectl label node worker2 node-role.kubernetes.io/worker=worker
        ```

3. Verify all system pods are running.

    ```bash
    guru@controller:~$ kubectl get pods -n kube-system
    ```

    You should see Calico, CoreDNS, etcd, kube-apiserver, kube-controller-manager, kube-proxy and kube-scheduler all in `Running` state.

4. Configure `kubectl` command-line completion for your current shell and persist it for future sessions.

    ```bash
    guru@controller:~$ source <(kubectl completion bash)
    guru@controller:~$ echo "source <(kubectl completion bash)" >> $HOME/.bashrc
    ```

5. Check whether the control-plane taint is present. In a training environment we remove it so pods can schedule on all nodes.

    ```bash
    guru@controller:~$ kubectl describe nodes | grep -i taint
    ```

    If you see `node-role.kubernetes.io/control-plane:NoSchedule` on the controller node, remove it:

    ```bash
    guru@controller:~$ kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    ```

    Verify both nodes now show `<none>` for taints:

    ```bash
    guru@controller:~$ kubectl describe nodes | grep -i taint
    Taints:             <none>
    Taints:             <none>
    Taints:             <none>
    ```

---

## Exercise 2.2: Explore kubectl and API Resources

1. Review the `kubectl` help output to become familiar with the available commands.

    ```bash
    guru@controller:~$ kubectl --help
    ```

2. Get a list of all available API resources and their API groups. Note that `pods` belongs to the stable `v1` group.

    ```bash
    guru@controller:~$ kubectl api-resources
    ```

    Example output (truncated):

    ```
    NAME                SHORTNAMES   APIVERSION   NAMESPACED   KIND
    configmaps          cm           v1           true         ConfigMap
    pods                po           v1           true         Pod
    services            svc          v1           true         Service
    deployments         deploy       apps/v1      true         Deployment
    replicasets         rs           apps/v1      true         ReplicaSet
    ```

3. Explore a specific sub-command using `--help`. For example, inspect the `taint` command.

    ```bash
    guru@controller:~$ kubectl taint --help
    ```

---

## Exercise 2.3: Create a Basic Pod

1. The lab files are located in `~/lfd459/ch02-architecture/`. Copy the files to your working directory.

    ```bash
    guru@controller:~$ cp ~/lfd459/ch02-architecture/* .
    guru@controller:~$ ls
    ```

    You should see: `basic.yaml`, `basic-later.yaml`, `basicservice.yaml`, `basicservice-AllDone.yaml`, `architecture-review1.yaml`

2. Review the minimal pod definition.

    ```bash
    guru@controller:~$ cat basic.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: basicpod
    spec:
      containers:
      - name: webcont
        image: nginx
    ```

3. Create the pod and verify it is running.

    ```bash
    guru@controller:~$ kubectl create -f basic.yaml
    pod/basicpod created

    guru@controller:~$ kubectl get pod
    NAME       READY   STATUS    RESTARTS   AGE
    basicpod   1/1     Running   0          23s
    ```

4. Inspect the pod details using `describe`. Look for the container image name and node assignment.

    ```bash
    guru@controller:~$ kubectl describe pod basicpod
    ```

5. Delete the pod and confirm it is gone.

    ```bash
    guru@controller:~$ kubectl delete pod basicpod
    pod "basicpod" deleted

    guru@controller:~$ kubectl get pod
    No resources found in default namespace.
    ```

6. Edit `basic.yaml` to expose port 80. Add two lines at the same indentation as `image:`.

    ```bash
    guru@controller:~$ vim basic.yaml
    ```

    The file should now look like this:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: basicpod
    spec:
      containers:
      - name: webcont
        image: nginx
        ports:
        - containerPort: 80
    ```

7. Re-create the pod and verify the internal IP address. Use `curl` to reach the nginx default page.

    ```bash
    guru@controller:~$ kubectl create -f basic.yaml
    pod/basicpod created

    guru@controller:~$ kubectl get pod -o wide
    NAME       READY   STATUS    RESTARTS   AGE   IP            NODE      ...
    basicpod   1/1     Running   0          18s   10.244.1.23   worker1   ...

    guru@controller:~$ curl http://10.244.1.23
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    ```

    !!! note
        The pod IP comes from the Calico CNI CIDR (`10.244.0.0/16`). The exact address will differ in your environment.

    Delete the pod before continuing.

    ```bash
    guru@controller:~$ kubectl delete pod basicpod
    ```

8. Review the service definition.

    ```bash
    guru@controller:~$ cat basicservice.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: basicservice
    spec:
      selector:
        type: webserver
      ports:
      - protocol: TCP
        port: 80
    ```

9. Edit `basic.yaml` to add a label that matches the service selector.

    ```bash
    guru@controller:~$ vim basic.yaml
    ```

    Add the `labels` section under `metadata`:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: basicpod
      labels:
        type: webserver     # <-- add this
    spec:
      containers:
      - name: webcont
        image: nginx
        ports:
        - containerPort: 80
    ```

10. Create both the pod and the service. Verify both are running.

    ```bash
    guru@controller:~$ kubectl create -f basic.yaml
    pod/basicpod created

    guru@controller:~$ kubectl create -f basicservice.yaml
    service/basicservice created

    guru@controller:~$ kubectl get pod
    NAME       READY   STATUS    RESTARTS   AGE
    basicpod   1/1     Running   0          110s

    guru@controller:~$ kubectl get svc
    NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    basicservice   ClusterIP   10.96.112.50    <none>        80/TCP    14s
    kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP   4h
    ```

11. Test access to the webserver via the service `CLUSTER-IP`.

    ```bash
    guru@controller:~$ curl http://10.96.112.50
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    ```

12. Now expose the service outside the cluster using `NodePort`. Delete the existing service and update the YAML.

    ```bash
    guru@controller:~$ kubectl delete svc basicservice
    service "basicservice" deleted
    ```

    Review `basicservice-AllDone.yaml` - it already has `type: NodePort` added:

    ```bash
    guru@controller:~$ cat basicservice-AllDone.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: basicservice
    spec:
      selector:
        type: webserver
      type: NodePort
      ports:
      - protocol: TCP
        port: 80
    ```

    Create the service using this file.

    ```bash
    guru@controller:~$ kubectl create -f basicservice-AllDone.yaml
    service/basicservice created

    guru@controller:~$ kubectl get svc
    NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
    basicservice   NodePort    10.100.139.155   <none>        80:31514/TCP   3s
    kubernetes     ClusterIP   10.96.0.1        <none>        443/TCP        47h
    ```

13. Note the high-numbered NodePort (e.g. `31514`). Test access from the controller node using any node's IP.

    ```bash
    guru@controller:~$ curl http://<worker1-ip>:31514
    <!DOCTYPE html>
    ...
    ```

    !!! tip
        Use `kubectl get nodes -o wide` to find the node IPs.

---

## Exercise 2.4: Multi-Container Pods

1. Edit `basic.yaml` to add a second container acting as a logging sidecar. Add the `fdlogger` container after the `webcont` container - the dash should align with the previous container dash.

    ```bash
    guru@controller:~$ vim basic.yaml
    ```

    ```yaml
    ...
    containers:
    - name: webcont
      image: nginx
      ports:
      - containerPort: 80
    - name: fdlogger
      image: fluentd
    ```

2. Delete and recreate the pod. This time `READY` should show `2/2`.

    ```bash
    guru@controller:~$ kubectl delete pod basicpod ; kubectl create -f basic.yaml
    pod "basicpod" deleted
    pod/basicpod created

    guru@controller:~$ kubectl get pod
    NAME       READY   STATUS    RESTARTS   AGE
    basicpod   2/2     Running   0          2m8s
    ```

3. Verify the `fdlogger` container is present in the describe output.

    ```bash
    guru@controller:~$ kubectl describe pod basicpod
    ```

    Look for the `fdlogger` section listing image, container ID and state.

4. Shut down the pod. We will revisit it with persistent storage in Chapter 5.

    ```bash
    guru@controller:~$ kubectl delete pod basicpod
    pod "basicpod" deleted
    ```

---

## Exercise 2.5: Create a Simple Deployment

1. Create a Deployment running the nginx webserver. A Deployment gives you scalability, self-healing, and rolling updates.

    ```bash
    guru@controller:~$ kubectl create deployment firstpod --image=nginx
    deployment.apps/firstpod created
    ```

2. Verify the Deployment and its managed pod are both running.

    ```bash
    guru@controller:~$ kubectl get deployment,pod
    NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/firstpod   1/1     1            1           10s

    NAME                            READY   STATUS    RESTARTS   AGE
    pod/firstpod-65c7f8b5bb-zmlp8   1/1     Running   0          10s
    ```

3. Describe the Deployment, then the Pod. Note the labels, replica count, strategy type, and the node the pod is scheduled on.

    ```bash
    guru@controller:~$ kubectl describe deployment firstpod
    guru@controller:~$ kubectl describe pod <pod-name>
    ```

    !!! tip
        Use Tab completion to avoid typing the full auto-generated pod name.

4. List available namespaces.

    ```bash
    guru@controller:~$ kubectl get namespaces
    NAME              STATUS   AGE
    default           Active   20m
    kube-node-lease   Active   20m
    kube-public       Active   20m
    kube-system       Active   20m
    ```

5. View all pods in the `kube-system` namespace.

    ```bash
    guru@controller:~$ kubectl get pod -n kube-system
    ```

    You should see Calico (`calico-system` or `kube-system`), CoreDNS, etcd, and other control plane components.

6. Query a namespace that does not exist. Note no error is returned - simply no resources found.

    ```bash
    guru@controller:~$ kubectl get pod -n fakenamespace
    No resources found in fakenamespaces namespace.
    ```

7. View pods across all namespaces at once.

    ```bash
    guru@controller:~$ kubectl get pod --all-namespaces
    ```

8. View multiple resource types in a single command. Note the short names: `deploy`, `rs`, `po`, `svc`, `ep`.

    ```bash
    guru@controller:~$ kubectl get deploy,rs,po,svc,ep
    ```

9. Delete the ReplicaSet and immediately check the resources again. You will see a **new** ReplicaSet and pod created automatically by the Deployment operator.

    ```bash
    guru@controller:~$ kubectl delete rs <replicaset-name>
    replicaset.apps "firstpod-65c7f8b5bb" deleted

    guru@controller:~$ kubectl get deploy,rs,po
    ```

    Notice the age on the new ReplicaSet and pod is only a few seconds - the Deployment reconciled immediately.

10. Delete the top-level Deployment. After about 30 seconds everything it manages cascades down.

    ```bash
    guru@controller:~$ kubectl delete deployment firstpod
    deployment.apps "firstpod" deleted

    guru@controller:~$ kubectl get deployment,rs,po,svc,ep
    ```

    Only the cluster services and endpoints remain.

11. Clean up the `basicservice` service.

    ```bash
    guru@controller:~$ kubectl delete svc basicservice
    service "basicservice" deleted
    ```

---

## Exercise 2.6: Domain Review

!!! warning "Important"
    The official CKAD curriculum and exam handbook can change at any time. **Always check the current version** before your exam.

1. Using a browser, visit [https://www.cncf.io/certification/ckad/](https://www.cncf.io/certification/ckad/) and read through the program description.

2. Open the **Curriculum Overview** and **Candidate Handbook** in separate tabs and read both in full.

3. In the Curriculum Overview, find the **Application Design and Build** domain. Notice the sub-topics - many will be covered in the upcoming chapters.

4. In the Candidate Handbook, find the **Important Instructions: CKA and CKAD** section. Note the Kubernetes version for the exam and the list of allowed resources.

5. Practice speed: using a timer, see how long it takes you to create and verify each of the following, then repeat to get faster:

    - A new pod with the `nginx` image, showing all containers running and `Ready` status.
    - A new service exposing the pod as a `NodePort`, with a working webserver reachable on the NodePort.
    - Update the pod to run the `nginx:1.28-alpine` image and re-verify access via NodePort.

6. Use the `architecture-review1.yaml` file to practice troubleshooting. Create the pod and determine why it does not start.

    ```bash
    guru@controller:~$ kubectl create -f architecture-review1.yaml
    guru@controller:~$ kubectl get pod
    guru@controller:~$ kubectl describe pod break1
    ```

    !!! hint
        Look carefully at the image name in the YAML. Use `kubectl describe` to read the event messages.

7. Once you have fixed and verified the pod, clean up before moving to the next chapter.

    ```bash
    guru@controller:~$ kubectl delete -f architecture-review1.yaml
    guru@controller:~$ kubectl delete pod basicpod --ignore-not-found
    guru@controller:~$ kubectl delete svc basicservice --ignore-not-found
    ```
