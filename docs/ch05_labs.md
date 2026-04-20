# Chapter 5 - Deployment Configuration

## Lab Overview

This chapter builds directly on Chapter 3. You will configure your existing `simpleapp` deployment with ConfigMaps (as environment variables and volume mounts), attach NFS-backed persistent storage, configure the fluentd logging sidecar with a ConfigMap, and practice rolling updates and rollbacks.

!!! note "Prerequisites"
    This chapter requires a working `~/app1/simpleapp.yaml` from Chapter 3, along with the local registry running (`try1` deployment with 6 replicas). If you need to rebuild it, revisit Chapter 3 before continuing.

    **Save a backup of your simpleapp.yaml before starting:**

    ```bash
cp ~/app1/simpleapp.yaml ~/beforeLab5.yaml
    ```

---

## Exercise 5.1: Configure the Deployment - ConfigMaps

ConfigMaps decouple configuration from container images. They can be created from literal values, files, or directories, and consumed as environment variables or volume mounts.

1. Connect to the **controller** node and move into the chapter lab directory.

    ```bash
    cd ~/lfd459/ch05-deployment-config
    ls
    ```

2. Create a directory of files to ingest into a ConfigMap, and a separate single file.

    ```bash
mkdir primary
echo c > primary/cyan
echo m > primary/magenta
echo y > primary/yellow
echo k > primary/black
echo "known as key" >> primary/black
echo blue > favorite
    ```

3. Create a ConfigMap called `colors` using all three ingestion methods at once: a literal, a single file, and a directory.

    ```bash
kubectl create configmap colors \
    --from-literal=text=black \
    --from-file=./favorite \
    --from-file=./primary/
    configmap/colors created
    ```

4. View the ConfigMap and verify all six keys are present.

    ```bash
kubectl get configmap colors -o yaml
    ```

    You should see keys: `black`, `cyan`, `favorite`, `magenta`, `text`, `yellow`.

5. Edit `~/app1/simpleapp.yaml` to add a single environment variable `ilike` sourced from the `colors` ConfigMap. Add the `env:` block inside the `simpleapp` container spec, after `imagePullPolicy`.

    ```bash
vim ~/app1/simpleapp.yaml
    ```

    Find the simpleapp container section and add:

    ```yaml
    - image: 10.97.40.62:5000/simpleapp
      imagePullPolicy: Always
      name: simpleapp
      env:
      - name: ilike
        valueFrom:
          configMapKeyRef:
            name: colors
            key: favorite
    ```

6. Delete and recreate the deployment.

    ```bash
kubectl delete deployment try1
kubectl create -f ~/app1/simpleapp.yaml
    ```

7. Verify the `ilike` environment variable is set inside the container.

    ```bash
kubectl get pods
kubectl exec -c simpleapp -it <try1-pod-name> \
    -- /bin/bash -c 'echo $ilike'
    blue
    ```

8. Edit `simpleapp.yaml` again to add `envFrom` so **all** keys from the `colors` ConfigMap are injected as environment variables.

    ```bash
vim ~/app1/simpleapp.yaml
    ```

    Add after `key: favorite`:

    ```yaml
          key: favorite
      envFrom:
      - configMapRef:
          name: colors
      imagePullPolicy: Always
    ```

9. Delete, recreate, and verify all color keys are visible as environment variables.

    ```bash
kubectl delete deployment try1
kubectl create -f ~/app1/simpleapp.yaml
kubectl exec -it <try1-pod-name> -- /bin/bash -c 'env' | grep -E 'ilike|cyan|magenta|yellow|black|text'
    ```

10. Create the `fast-car` ConfigMap from the provided YAML file.

    ```bash
kubectl create -f car-map.yaml
    configmap/fast-car created
kubectl get configmap fast-car -o yaml
    ```

11. Edit `simpleapp.yaml` to mount the `fast-car` ConfigMap as a volume at `/etc/cars` inside the `simpleapp` container.

    ```bash
vim ~/app1/simpleapp.yaml
    ```

    In the `simpleapp` container spec, add before `env:`:

    ```yaml
        volumeMounts:
        - mountPath: /etc/cars
          name: car-vol
    ```

    At the end of the pod `spec` (after all containers, before `status:`), add:

    ```yaml
      volumes:
      - name: car-vol
        configMap:
          defaultMode: 420
          name: fast-car
    ```

    !!! tip
        The `simpleapp.yaml-with-edits` file in your home directory shows the complete final state of this file after all edits in this chapter. Use it as a reference if needed.

12. Delete and recreate the deployment.

    ```bash
kubectl delete deployment try1
kubectl create -f ~/app1/simpleapp.yaml
    ```

13. The deployment will show `0/6` ready because the `readinessProbe` is still checking for `/tmp/healthy`. Update the probe to check for `/etc/cars` instead.

    ```bash
kubectl delete deployment try1
vim ~/app1/simpleapp.yaml
    ```

    Change the `readinessProbe` exec command inside the `simpleapp` container:

    ```yaml
        readinessProbe:
          exec:
            command:
            - ls
            - /etc/cars
          periodSeconds: 5
    ```

    ```bash
kubectl create -f ~/app1/simpleapp.yaml
    ```

14. Wait about a minute. All six pods should show `2/2 Running`.

    ```bash
kubectl get pods
    NAME                         READY   STATUS    RESTARTS   AGE
    try1-7865dcb948-2dzc8        2/2     Running   0          1m
    try1-7865dcb948-7fkh7        2/2     Running   0          1m
    ...
    ```

15. Verify the ConfigMap data is visible inside the container as a file.

    ```bash
kubectl exec -c simpleapp -it <try1-pod-name> \
    -- /bin/bash -c 'cat /etc/cars/car.trim'
    Shelby
    ```

---

## Exercise 5.2: Configure the Deployment - Attaching Storage

We will configure an NFS server on the controller node, create a PersistentVolume backed by it, and attach it to the `try1` deployment.

1. Run the `CreateNFS.sh` script to install and configure NFS on the **controller** node.

    ```bash
bash CreateNFS.sh
    ```

    The script installs `nfs-kernel-server`, creates `/opt/sfw/`, exports it, and creates `/opt/sfw/hello.txt`. At the end you should see:

    ```
    Should be ready. Test here and second node
    Export list for localhost:
    /opt/sfw *
    ```

2. Connect to **worker1** and verify the NFS export is visible from the worker.

    ```bash
sudo apt-get -y install nfs-common nfs-kernel-server
showmount -e controller
    Export list for controller:
    /opt/sfw *
    ```

    Repeat on **worker2**.

3. Test the mount from worker1.

    ```bash
sudo mount controller:/opt/sfw /mnt
ls -l /mnt
    total 4
    -rw-r--r-- 1 root root 9 ... hello.txt
sudo umount /mnt
    ```

4. Return to the **controller**. Edit `PVol.yaml` - the `server:` field currently says `cp`, change it to `controller`.

    ```bash
vim PVol.yaml
    ```

    ```yaml
    nfs:
      path: /opt/sfw
      server: controller    # <-- change from cp to controller
      readOnly: false
    ```

5. Create the PersistentVolume and verify it shows `Available`.

    ```bash
kubectl create -f PVol.yaml
    persistentvolume/pvvol-1 created
kubectl get pv
    NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
    pvvol-1    1Gi        RWX            Retain           Available
    ...
    ```

6. Check any existing PVCs (from the registry deployment in Chapter 3).

    ```bash
kubectl get pvc
    ```

7. Create a PVC using `pvc.yaml`.

    ```bash
kubectl create -f pvc.yaml
    persistentvolumeclaim/pvc-one created
kubectl get pvc pvc-one
    NAME      STATUS   VOLUME    CAPACITY   ACCESS MODES
    pvc-one   Bound    pvvol-1   1Gi        RWX
    ```

    Note the PVC claimed 1Gi even though only 200Mi was requested - Kubernetes uses the smallest available volume that satisfies the request.

8. Verify `pvvol-1` is now `Bound`.

    ```bash
kubectl get pv pvvol-1
    NAME      CAPACITY   ACCESS MODES   STATUS   CLAIM
    pvvol-1   1Gi        RWX            Bound    default/pvc-one
    ```

9. Edit `~/app1/simpleapp.yaml` to add the NFS volume to the `simpleapp` container.

    ```bash
vim ~/app1/simpleapp.yaml
    ```

    In the simpleapp `volumeMounts` section, add after the `car-vol` mount:

    ```yaml
        - name: nfs-vol
          mountPath: /opt
    ```

    In the `volumes` section, add after the `car-vol` volume:

    ```yaml
      - name: nfs-vol
        persistentVolumeClaim:
          claimName: pvc-one
    ```

10. Delete and recreate the deployment.

    ```bash
kubectl delete deployment try1
kubectl create -f ~/app1/simpleapp.yaml
    ```

11. Verify the NFS volume is mounted under `/opt` in a pod.

    ```bash
kubectl describe pod <try1-pod-name> | grep -A5 "Mounts:"
    Mounts:
      /etc/cars from car-vol (rw)
      /opt from nfs-vol (rw)
      ...
    ```

---

## Exercise 5.3: Using ConfigMaps to Configure Containers

Now we return to the `basicpod` from Chapter 2 and fully configure the fluentd logging sidecar using a ConfigMap and a shared PersistentVolume.

1. Review the current `basic.yaml` in your home directory.

    ```bash
cat basic.yaml
    ```

    It should have two containers: `webcont` (nginx) and `fdlogger` (fluentd).

2. Create the directory that will back the hostPath PV.

    ```bash
sudo mkdir /tmp/weblog
    ```

3. Create the PV using `weblog-pv.yaml`.

    ```bash
kubectl create -f weblog-pv.yaml
    persistentvolume/weblog-pv-volume created
kubectl get pv weblog-pv-volume
    NAME               CAPACITY   ACCESS MODES   STATUS
    weblog-pv-volume   100Mi      RWO            Available
    ```

4. Create the PVC using `weblog-pvc.yaml` and verify it binds to `weblog-pv-volume`.

    ```bash
kubectl create -f weblog-pvc.yaml
    persistentvolumeclaim/weblog-pv-claim created
kubectl get pvc weblog-pv-claim
    NAME             STATUS   VOLUME             CAPACITY   STORAGE CLASS
    weblog-pv-claim  Bound    weblog-pv-volume   100Mi      manual
    ```

5. Edit `basic.yaml` to add the shared volume to both containers.

    ```bash
vim basic.yaml
    ```

    The final file should look like `basic.yaml-with-edits`:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: basicpod
      labels:
        type: webserver
    spec:
      volumes:
      - name: weblog-pv-storage
        persistentVolumeClaim:
          claimName: weblog-pv-claim
      containers:
      - name: webcont
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/var/log/nginx/"
          name: weblog-pv-storage
      - name: fdlogger
        image: fluent/fluentd:v1.16
        volumeMounts:
        - mountPath: "/var/log"
          name: weblog-pv-storage
    ```

6. Create the pod and open a shell into the `webcont` container. Verify the nginx access log is a real file and start tailing it.

    ```bash
kubectl create -f basic.yaml
    pod/basicpod created
kubectl exec -c webcont -it basicpod -- /bin/bash
    root@basicpod:/# ls -l /var/log/nginx/access.log
    -rw-r--r-- 1 root root 0 ... /var/log/nginx/access.log

    root@basicpod:/# tail -f /var/log/nginx/access.log
    ```

7. Open a **second terminal** on the controller. Get the pod IP and curl the nginx default page.

    ```bash
kubectl get pods -o wide
    NAME       READY   STATUS    IP               NODE
    basicpod   2/2     Running   10.244.1.23      worker1
curl http://10.244.1.23
    ```

    You should see a new log entry appear in the tail window in your first terminal. Use `Ctrl-C` and `exit` to return to the host.

8. Create the fluentd ConfigMap from `weblog-configmap.yaml`.

    ```bash
kubectl create -f weblog-configmap.yaml
    configmap/fluentd-config created
    ```

9. Check the current logs for both containers.

    ```bash
kubectl logs basicpod webcont
kubectl logs basicpod fdlogger
    ```

10. Edit `basic.yaml` to configure the `fdlogger` container to use the ConfigMap.

    ```bash
vim basic.yaml
    ```

    In the `volumes` section, add after `weblog-pv-storage`:

    ```yaml
      - name: log-config
        configMap:
          name: fluentd-config
    ```

    In the `fdlogger` container, add `env` and a second `volumeMounts` entry:

    ```yaml
      - name: fdlogger
        image: fluent/fluentd:v1.16
        env:
        - name: FLUENTD_OPT
          value: -c /fluentd/etc/fluent.conf
        volumeMounts:
        - mountPath: "/var/log"
          name: weblog-pv-storage
        - name: log-config
          mountPath: "/fluentd/etc"
    ```

11. Delete and recreate the pod.

    ```bash
kubectl delete pod basicpod
kubectl create -f basic.yaml
kubectl get pod basicpod -o wide
    NAME       READY   STATUS    IP
    basicpod   2/2     Running   10.244.1.xx
    ```

12. Curl the nginx page a few times, then check the `fdlogger` logs.

    ```bash
curl http://<basicpod-ip>
curl http://<basicpod-ip>
kubectl logs basicpod fdlogger | tail -10
    ```

    Look for lines like:

    ```
    count.format1: {"message":"10.244.x.x - - [date] \"GET / HTTP/1.1\" 200 612 ..."}
    ```

---

## Exercise 5.4: Rolling Updates and Rollbacks

1. Move into the `app1` directory and add a comment to `simple.py` to produce a slightly different image.

    ```bash
cd ~/app1
vim simple.py
    ```

    Add a comment on the last line:

    ```python
    ## Sleep for five seconds then continue the loop
    time.sleep(5)
    ## Adding a new comment so image is different.
    ```

2. Check the current images.

    ```bash
sudo podman images | grep simple
    ```

3. Build the updated image.

    ```bash
sudo podman build -t simpleapp .
    ```

4. Tag and push the updated image as `v2`.

    ```bash
sudo podman tag simpleapp $repo/simpleapp:v2
sudo podman push $repo/simpleapp:v2
    ```

5. Verify the registry now has both `latest` and `v2` tagged images.

    ```bash
curl $repo/v2/simpleapp/tags/list
    {"name":"simpleapp","tags":["latest","v2"]}
    ```

6. Connect to **worker1** and pull both versions to confirm the worker can access them.

    ```bash
sudo podman pull $repo/simpleapp
sudo podman pull $repo/simpleapp:v2
    ```

7. Return to the **controller**. Use `kubectl edit` to update the `try1` deployment to use `v2`.

    ```bash
kubectl edit deployment try1
    ```

    Change:

    ```yaml
    - image: 10.97.40.62:5000/simpleapp
    ```

    To:

    ```yaml
    - image: 10.97.40.62:5000/simpleapp:v2
    ```

8. Watch the rolling update.

    ```bash
kubectl get events | tail -20
kubectl get pods
    ```

9. Verify the new pods are using `v2`.

    ```bash
kubectl describe pod <try1-pod-name> | grep Image:
    Image:    10.97.40.62:5000/simpleapp:v2
    Image:    registry.k8s.io/goproxy:0.1
    ```

10. View the rollout history.

    ```bash
kubectl rollout history deployment try1
    REVISION   CHANGE-CAUSE
    1          <none>
    2          <none>
    ```

11. Compare the two revisions by saving them to files and diffing.

    ```bash
kubectl rollout history deployment try1 --revision=1 > one.out
kubectl rollout history deployment try1 --revision=2 > two.out
diff one.out two.out
    ```

12. Preview the rollback without applying it.

    ```bash
kubectl rollout undo --dry-run=client deployment/try1
    ```

13. Roll back to revision 1.

    ```bash
kubectl rollout undo deployment try1 --to-revision=1
    deployment.apps/try1 rolled back
    ```

14. Wait for all pods to cycle and verify they are back to the original image.

    ```bash
kubectl get pods
kubectl describe pod <try1-pod-name> | grep Image:
    Image:    10.97.40.62:5000/simpleapp
    ```

---

## Exercise 5.5: Domain Review

!!! important
    Source pages and content in this review can change at any time. Always check current information before your exam.

Revisit the CKAD curriculum for topics covered in this chapter:

- Utilize persistent and ephemeral volumes
- Understand Deployments and how to perform rolling updates
- Understand ConfigMaps

- Create a new secret called `specialofday` using the key `entree` and the value `meatloaf`.
- Create a new deployment called `foodie` running the `nginx` image.
- Add the `specialofday` secret to the `foodie` pod, mounted as a volume under `/food/`.
- Exec a bash shell into a `foodie` pod and verify the secret is correctly mounted.
- Update the `foodie` deployment to use the `nginx:1.28.1-alpine` image and verify the new image is in use.
- Roll back the deployment and verify the standard nginx image is running again.
- Create a new 200M NFS PersistentVolume called `reviewvol` using the NFS server configured earlier (`controller:/opt/sfw`).
- Create a new PVC called `reviewpvc` that binds to `reviewvol`.
- Edit the `foodie` deployment to mount the PVC under `/newvol`.
- Exec into the nginx container and verify the volume is mounted.
- Delete all resources created during this review.

    ```bash
kubectl delete deployment foodie --ignore-not-found
kubectl delete secret specialofday --ignore-not-found
kubectl delete pvc reviewpvc --ignore-not-found
kubectl delete pv reviewvol --ignore-not-found
    ```
