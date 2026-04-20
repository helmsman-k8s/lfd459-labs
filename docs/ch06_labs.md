# Chapter 6 ? Understanding Security

## Lab Overview

In this chapter you will work with security contexts to control container UIDs and Linux capabilities, create and consume Secrets, configure ServiceAccounts with RBAC, and understand NetworkPolicies. A new working directory `~/app2` is used throughout.

!!! note "NetworkPolicy and Calico"
    This cluster uses **Calico** as the CNI plugin. Calico **fully enforces NetworkPolicies** ? when you apply a policy in Exercises 6.4 and 6.5, traffic will actually be blocked as described. The curl and nc tests will time out as expected. This matches the behaviour you will see on the CKAD exam.

---

## Exercise 6.1: Set SecurityContext for a Pod and Container

A SecurityContext restricts what a container process can do ? which UID it runs as, whether it can escalate privileges, and which Linux capabilities it has.

1. Connect to the **controller** node. Create a new directory for the second application and copy the chapter lab files.

    ```bash
    guru@controller:~$ mkdir ~/app2
    guru@controller:~$ cp ~/lfd459/ch06-security/* ~/app2/
    guru@controller:~$ cd ~/app2
    ```

2. Review and create the `secondapp` pod. It sets a pod-level UID of 1000 and a container-level UID of 2000. The container setting takes precedence.

    ```bash
    guru@controller:~/app2$ cat second.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: secondapp
    spec:
      securityContext:
        runAsUser: 1000
      containers:
      - name: busy
        image: busybox
        command:
          - sleep
          - "3600"
        securityContext:
          runAsUser: 2000
          allowPrivilegeEscalation: false
    ```

    ```bash
    guru@controller:~/app2$ kubectl create -f second.yaml
    pod/secondapp created

    guru@controller:~/app2$ kubectl get pod secondapp
    NAME         READY   STATUS    RESTARTS   AGE
    secondapp    1/1     Running   0          21s
    ```

3. Compare the pod YAML output with a deployment. Note the difference in the `status` section.

    ```bash
    guru@controller:~/app2$ kubectl get pod secondapp -o yaml
    ```

4. Exec into the container and verify the process is running as UID 2000 (the container setting, not the pod's 1000).

    ```bash
    guru@controller:~/app2$ kubectl exec -it secondapp -- sh
    ```

    Inside the container:

    ```sh
    / $ ps aux
    PID   USER     COMMAND
    1     2000     sleep 3600
    8     2000     sh
    ```

5. Check the current Linux capability bitmask for PID 1.

    ```sh
    / $ grep Cap /proc/1/status
    CapBnd: 00000000a80425fb
    / $ exit
    ```

6. Decode the capability bitmask using `capsh` on the controller. Note the list of capabilities granted.

    ```bash
    guru@controller:~/app2$ capsh --decode=00000000a80425fb
    ```

    You should see approximately 14 capabilities including `cap_chown`, `cap_net_bind_service`, etc.

7. Delete the pod and edit `second.yaml` to add two additional Linux capabilities to the container: `NET_ADMIN` and `SYS_TIME`.

    ```bash
    guru@controller:~/app2$ kubectl delete pod secondapp
    guru@controller:~/app2$ vim second.yaml
    ```

    Add the `capabilities` block inside the container's `securityContext`:

    ```yaml
        securityContext:
          runAsUser: 2000
          allowPrivilegeEscalation: false
          capabilities:
            add: ["NET_ADMIN", "SYS_TIME"]
    ```

8. Recreate the pod and check the new capability bitmask.

    ```bash
    guru@controller:~/app2$ kubectl create -f second.yaml
    guru@controller:~/app2$ kubectl exec -it secondapp -- sh
    ```

    Inside:

    ```sh
    / $ grep Cap /proc/1/status
    CapBnd: 00000000aa0435fb
    / $ exit
    ```

9. Decode the new bitmask. Confirm `cap_net_admin` and `cap_sys_time` are now present (16 capabilities total).

    ```bash
    guru@controller:~/app2$ capsh --decode=00000000aa0435fb
    ```

---

## Exercise 6.2: Create and Consume Secrets

Secrets store sensitive data in base64-encoded form. They are consumed like ConfigMaps ? as environment variables or volume mounts.

1. Generate a base64-encoded password.

    ```bash
    guru@controller:~/app2$ echo LFTr@1n | base64
    TEZUckAxbgo=
    ```

2. Review `secret.yaml` ? it already contains the encoded password.

    ```bash
    guru@controller:~/app2$ cat secret.yaml
    ```

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: lfsecret
    data:
      password: TEZUckAxbgo=
    ```

3. Create the secret.

    ```bash
    guru@controller:~/app2$ kubectl create -f secret.yaml
    secret/lfsecret created
    ```

4. Edit `second.yaml` to mount the secret as a volume at `/mysqlpassword`. Add `volumeMounts` to the container and `volumes` at the pod spec level.

    ```bash
    guru@controller:~/app2$ vim second.yaml
    ```

    In the container spec (after the `capabilities` block):

    ```yaml
          volumeMounts:
          - name: mysql
            mountPath: /mysqlpassword
    ```

    At the pod spec level (same depth as `containers:`):

    ```yaml
      volumes:
      - name: mysql
        secret:
          secretName: lfsecret
    ```

5. Delete, recreate, and verify the secret is accessible inside the container in decoded form.

    ```bash
    guru@controller:~/app2$ kubectl delete pod secondapp
    guru@controller:~/app2$ kubectl create -f second.yaml
    guru@controller:~/app2$ kubectl get pod secondapp
    NAME         READY   STATUS    RESTARTS   AGE
    secondapp    1/1     Running   0          34s

    guru@controller:~/app2$ kubectl exec -ti secondapp -- /bin/sh
    ```

    Inside:

    ```sh
    / $ cat /mysqlpassword/password
    LFTr@1n

    / $ ls -al /mysqlpassword/
    total 4
    drwxrwxrwt  ...  .
    dr-xr-xr-x  ...  ..
    lrwxrwxrwx  ...  ..data -> ..2026_03_19_16_06_41.694089911
    lrwxrwxrwx  ...  password -> ..data/password

    / $ exit
    ```

    The mount point uses symlinks ? the actual data is in a timestamped directory, and `..data` is a symlink to it. This allows Kubernetes to atomically update secrets.

---

## Exercise 6.3: Working with ServiceAccounts

ServiceAccounts provide an identity for pod processes to interact with the Kubernetes API. We will create one with read access to secrets.

1. Return to the home directory and view existing secrets.

    ```bash
    guru@controller:~/app2$ cd
    guru@controller:~$ kubectl get secrets
    guru@controller:~$ kubectl get secrets --all-namespaces
    ```

2. Create the ServiceAccount from `serviceaccount.yaml`.

    ```bash
    guru@controller:~$ kubectl create -f ~/app2/serviceaccount.yaml
    serviceaccount/secret-access-sa created

    guru@controller:~$ kubectl get serviceaccounts
    NAME               SECRETS   AGE
    default            0         ...
    secret-access-sa   0         34s
    ```

3. View existing ClusterRoles and compare `admin` vs `cluster-admin`.

    ```bash
    guru@controller:~$ kubectl get clusterroles
    guru@controller:~$ kubectl get clusterroles admin -o yaml
    guru@controller:~$ kubectl get clusterroles cluster-admin -o yaml
    ```

    Note that `cluster-admin` uses `*` (wildcard) for all resources and verbs.

4. Create the `secret-access-cr` ClusterRole from `clusterrole.yaml`. It grants only `get` and `list` on secrets.

    ```bash
    guru@controller:~$ kubectl create -f ~/app2/clusterrole.yaml
    clusterrole.rbac.authorization.k8s.io/secret-access-cr created

    guru@controller:~$ kubectl get clusterrole secret-access-cr -o yaml
    ```

5. Bind the ClusterRole to the ServiceAccount using `rolebinding.yaml`.

    ```bash
    guru@controller:~$ kubectl create -f ~/app2/rolebinding.yaml
    rolebinding.rbac.authorization.k8s.io/secret-rb created

    guru@controller:~$ kubectl get rolebindings
    NAME        AGE
    secret-rb   17s
    ```

6. Verify the current `secondapp` pod uses the `default` serviceAccount.

    ```bash
    guru@controller:~$ kubectl get pod secondapp -o yaml | grep serviceAccount
    serviceAccount: default
    serviceAccountName: default
    ```

7. Edit `~/app2/second.yaml` and add `serviceAccountName: secret-access-sa` to the pod spec, before `securityContext`.

    ```bash
    guru@controller:~$ vim ~/app2/second.yaml
    ```

    ```yaml
    spec:
      serviceAccountName: secret-access-sa   # <-- add this line
      securityContext:
        runAsUser: 1000
    ```

8. Delete and recreate. Verify the new serviceAccount is in use.

    ```bash
    guru@controller:~$ kubectl delete pod secondapp
    guru@controller:~$ kubectl create -f ~/app2/second.yaml
    guru@controller:~$ kubectl get pod secondapp -o yaml | grep serviceAccount
    serviceAccount: secret-access-sa
    serviceAccountName: secret-access-sa
    ```

---

## Exercise 6.4: Implement a NetworkPolicy

!!! note "Calico enforces NetworkPolicies"
    This cluster uses Calico, which fully enforces NetworkPolicies. Once you apply the policy in the next exercise, traffic will be blocked as described. The tests will time out as expected.

1. Review `allclosed.yaml`. This deny-all policy matches every pod (`podSelector: {}`) and blocks both ingress and egress.

    ```bash
    guru@controller:~/app2$ cat ~/app2/allclosed.yaml
    ```

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: deny-default
    spec:
      podSelector: {}
      policyTypes:
      - Ingress
      - Egress
    ```

2. Before applying the policy, prepare the `secondapp` pod with a second container (`webserver` running nginx) so we have something to test ingress against. First delete the existing pod.

    ```bash
    guru@controller:~/app2$ kubectl delete pod secondapp
    ```

    Edit `second.yaml` to add the nginx webserver container **before** the busy container, and comment out the pod-level `securityContext` (nginx requires root to start):

    ```bash
    guru@controller:~/app2$ vim second.yaml
    ```

    ```yaml
    spec:
      serviceAccountName: secret-access-sa
    # securityContext:        # <-- comment out
    #   runAsUser: 1000       # <-- comment out
      containers:
      - name: webserver
        image: nginx          # <-- add this and following line
      - name: busy
        image: busybox
        command:
    ```

3. Create the pod. Both containers should start (you may see a brief `CrashLoopBackOff` for `busy` while it pulls).

    ```bash
    guru@controller:~/app2$ kubectl create -f second.yaml
    guru@controller:~/app2$ kubectl get pods
    NAME        READY   STATUS    RESTARTS   AGE
    secondapp   2/2     Running   0          5s
    ```

    !!! note "If only 1/2 containers start"
        If the pod shows `CrashLoopBackOff`, check `kubectl logs secondapp webserver`. If there is a permission error about creating directories, ensure the pod-level `securityContext.runAsUser` is commented out ? nginx must run as root.

4. Try to expose the pod ? this will fail because the pod has no labels.

    ```bash
    guru@controller:~/app2$ kubectl expose pod secondapp --type=NodePort --port=80
    error: couldn't retrieve selectors via --selector flag: the pod has no labels
    ```

5. Add a label to the pod. Edit `second.yaml` to add `labels:` in the metadata section, then delete and recreate.

    ```bash
    guru@controller:~/app2$ vim second.yaml
    ```

    ```yaml
    metadata:
      name: secondapp
      labels:
        example: second    # <-- add this
    ```

    ```bash
    guru@controller:~/app2$ kubectl delete pod secondapp
    guru@controller:~/app2$ kubectl create -f second.yaml
    ```

6. Create the NodePort service and set it to port 32000.

    ```bash
    guru@controller:~/app2$ kubectl create service nodeport secondapp --tcp=80
    service/secondapp created
    ```

    Edit the service to change the selector to `example: second` and set the nodePort to `32000`.

    ```bash
    guru@controller:~/app2$ kubectl edit svc secondapp
    ```

    Change:
    ```yaml
    nodePort: <random>   # set to 32000
    selector:
      example: second    # change from app: secondapp
    ```

7. Verify the service and test access.

    ```bash
    guru@controller:~/app2$ kubectl get svc secondapp
    NAME        TYPE       CLUSTER-IP      PORT(S)
    secondapp   NodePort   10.97.96.75     80:32000/TCP

    guru@controller:~/app2$ curl http://10.97.96.75
    ```

    You should see the nginx welcome page.

8. Test egress from within the container.

    ```bash
    guru@controller:~/app2$ kubectl exec -it -c busy secondapp -- sh
    ```

    Inside:

    ```sh
    / $ nc -vz 127.0.0.1 80
    127.0.0.1 (127.0.0.1:80) open

    / $ nc -vz www.linux.com 80
    www.linux.com (151.101.185.5:80) open

    / $ exit
    ```

---

## Exercise 6.5: Testing the NetworkPolicy

!!! note "Traffic will be blocked"
    With Calico, the policy below will be enforced. Expect curl and nc tests to time out once the deny-all policy is applied.

1. Apply the deny-all policy.

    ```bash
    guru@controller:~/app2$ kubectl create -f ~/app2/allclosed.yaml
    networkpolicy.networking.k8s.io/deny-default created
    ```

2. Test that traffic is now blocked. Both the NodePort and egress to the internet should time out (Ctrl-C after ~10 seconds if needed).

    ```bash
    # From the controller to the service ClusterIP (should time out)
    guru@controller:~/app2$ curl http://10.97.96.75

    # From inside the container to the internet (should time out)
    guru@controller:~/app2$ kubectl exec -it -c busy secondapp -- sh
    / $ nc -vz www.linux.com 80
    / $ exit
    ```

3. Update the policy to remove the Egress restriction (comment it out) and replace.

    ```bash
    guru@controller:~/app2$ vim ~/app2/allclosed.yaml
    ```

    ```yaml
    spec:
      podSelector: {}
      policyTypes:
      - Ingress
    # - Egress    # <-- comment out
    ```

    ```bash
    guru@controller:~/app2$ kubectl replace -f ~/app2/allclosed.yaml
    networkpolicy.networking.k8s.io/deny-default replaced
    ```

4. Get the pod IP, then add an ingress rule to allow traffic from any pod.

    ```bash
    guru@controller:~/app2$ kubectl get pod secondapp -o wide
    ```

    Edit `allclosed.yaml` to add an ingress rule:

    ```yaml
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector: {}
    # - Egress
    ```

    ```bash
    guru@controller:~/app2$ kubectl replace -f ~/app2/allclosed.yaml
    ```

5. Test pod-to-pod ingress with ping using a temporary alpine pod.

    ```bash
    guru@controller:~/app2$ kubectl run -it test --rm=true --image alpine \
    -- ping -c5 <secondapp-pod-ip>
    ```

    With Calico this will succeed ? the ingress rule allows all pod traffic including ICMP. The next step will restrict to TCP port 80 only, at which point ping will fail.

6. Update the policy to only allow TCP port 80, and block ICMP (ping).

    ```bash
    guru@controller:~/app2$ vim ~/app2/allclosed.yaml
    ```

    ```yaml
    ingress:
    - from:
      - podSelector: {}
      ports:
      - port: 80
        protocol: TCP
    ```

    ```bash
    guru@controller:~/app2$ kubectl replace -f ~/app2/allclosed.yaml
    ```

    ```bash
    # On Calico this would succeed (HTTP allowed):
    guru@controller:~/app2$ curl http://10.97.96.75

    # On Calico this would fail (ICMP not in the policy):
    guru@controller:~/app2$ kubectl run -it test --rm=true --image alpine \
    -- ping -c5 <secondapp-pod-ip>
    ```

7. Delete the NetworkPolicy so the registry and other pods remain accessible for future chapters.

    ```bash
    guru@controller:~/app2$ kubectl delete networkpolicies deny-default
    networkpolicy.networking.k8s.io "deny-default" deleted
    ```

---

## Exercise 6.6: Domain Review

!!! warning "Important"
    Source pages and content in this review can change at any time. Always check current information before your exam.

Revisit the CKAD curriculum for topics covered in this chapter:

- Understand authentication, authorization, and admission control
- Create and consume Secrets
- Understand ServiceAccounts
- Understand SecurityContexts
- Demonstrate basic understanding of NetworkPolicies

1. Create a new deployment using the `nginx` image.

2. Create a `LoadBalancer` service to expose the deployment on port 80. Note the `EXTERNAL-IP` will remain `<pending>` without a cloud provider ? the NodePort still works.

3. Create a new NetworkPolicy called `netblock` that blocks all traffic to pods in this deployment only (use a `podSelector` that matches your deployment's labels, not `{}`). Verify the object is created.

4. Update `netblock` to allow ingress traffic on port 80 only. Verify with curl.

5. Create a pod using `security-review1.yaml`.

    ```bash
    guru@controller:~$ kubectl create -f ~/app2/security-review1.yaml
    ```

6. Check the pod status.

    ```bash
    guru@controller:~$ kubectl get pod securityreview
    guru@controller:~$ kubectl describe pod securityreview
    guru@controller:~$ kubectl logs securityreview
    ```

    The pod will fail because nginx tries to run as root but `runAsUser: 3000` is set at the container level.

7. Exec into the container (or use `kubectl debug`) to find the UID that nginx actually needs. Hint: check `/etc/passwd` inside the nginx image for the `nginx` user's UID.

8. Edit `security-review1.yaml` and set the container `runAsUser` to the correct nginx UID. Recreate the pod and verify it runs.

9. Create a new ServiceAccount called `securityaccount`.

    ```bash
    guru@controller:~$ kubectl create serviceaccount securityaccount
    ```

10. Create a ClusterRole named `secrole` that allows `create`, `delete`, and `list` of pods in all apiGroups.

11. Bind `secrole` to `securityaccount`.

12. Create a token for `securityaccount` and save only the token value to `/tmp/securitytoken`.

    ```bash
    guru@controller:~$ kubectl create token securityaccount
    ```

    Copy the output string (starts with `eyJh...`) into the file:

    ```bash
    guru@controller:~$ kubectl create token securityaccount > /tmp/securitytoken
    ```

13. Clean up all resources created during this review.

    ```bash
    guru@controller:~$ kubectl delete deployment nginx --ignore-not-found
    guru@controller:~$ kubectl delete svc nginx --ignore-not-found
    guru@controller:~$ kubectl delete networkpolicy netblock --ignore-not-found
    guru@controller:~$ kubectl delete pod securityreview --ignore-not-found
    guru@controller:~$ kubectl delete serviceaccount securityaccount --ignore-not-found
    guru@controller:~$ kubectl delete clusterrole secrole --ignore-not-found
    guru@controller:~$ kubectl delete clusterrolebinding secrole --ignore-not-found
    ```
