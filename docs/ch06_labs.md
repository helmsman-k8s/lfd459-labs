# Chapter 6 - Understanding Security

## Lab Overview

In this chapter you will work with security contexts to control container UIDs and Linux capabilities, create and consume Secrets, configure ServiceAccounts with RBAC, and understand NetworkPolicies. A new working directory `~/app2` is used throughout.

!!! note "NetworkPolicy and Calico"
    This cluster uses **Calico** as the CNI plugin. Calico **fully enforces NetworkPolicies** - when you apply a policy in Exercises 6.4 and 6.5, traffic will actually be blocked as described. The curl and nc tests will time out as expected. This matches the behaviour you will see on the CKAD exam.

---

## Exercise 6.1: Set SecurityContext for a Pod and Container

A SecurityContext restricts what a container process can do - which UID it runs as, whether it can escalate privileges, and which Linux capabilities it has.

1. Connect to the **controller** node. Move into the chapter lab directory. A new working directory `~/lfd459/ch06-security` is used throughout.

    ```bash
    cd ~/lfd459/ch06-security
    ```

2. Review and create the `secondapp` pod. It sets a pod-level UID of 1000 and a container-level UID of 2000. The container setting takes precedence.

    ```bash
    cat second.yaml
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
    kubectl create -f second.yaml
    ```

    ```
    pod/secondapp created
    ```

    ```bash
    kubectl get pod secondapp
    ```

    ```
    NAME         READY   STATUS    RESTARTS   AGE
    secondapp    1/1     Running   0          21s
    ```

3. Exec into the container and verify the process is running as UID 2000.

    ```bash
    kubectl exec -it secondapp -- sh
    ```

    Inside the container:

    ```
    / $ ps aux
    PID   USER     COMMAND
    1     2000     sleep 3600
    8     2000     sh
    ```

4. Check the current Linux capability bitmask for PID 1.

    ```
    / $ grep Cap /proc/1/status
    CapBnd: 00000000a80425fb
    / $ exit
    ```

5. Decode the capability bitmask using `capsh` on the controller.

    ```bash
    capsh --decode=00000000a80425fb
    ```

    You should see approximately 14 capabilities including `cap_chown`, `cap_net_bind_service`, etc.

6. Delete the pod and edit `second.yaml` to add two additional Linux capabilities: `NET_ADMIN` and `SYS_TIME`.

    ```bash
    kubectl delete pod secondapp
    vim second.yaml
    ```

    Add the `capabilities` block inside the container's `securityContext`:

    ```yaml
        securityContext:
          runAsUser: 2000
          allowPrivilegeEscalation: false
          capabilities:
            add: ["NET_ADMIN", "SYS_TIME"]
    ```

7. Recreate the pod and check the new capability bitmask.

    ```bash
    kubectl create -f second.yaml
    kubectl exec -it secondapp -- sh
    ```

    Inside:

    ```
    / $ grep Cap /proc/1/status
    CapBnd: 00000000aa0435fb
    / $ exit
    ```

8. Decode the new bitmask. Confirm `cap_net_admin` and `cap_sys_time` are now present (16 capabilities total).

    ```bash
    capsh --decode=00000000aa0435fb
    ```

---

## Exercise 6.2: Create and Consume Secrets

Secrets store sensitive data in base64-encoded form. They are consumed like ConfigMaps - as environment variables or volume mounts.

1. Generate a base64-encoded password.

    ```bash
    echo LFTr@1n | base64
    ```

    ```
    TEZUckAxbgo=
    ```

2. Review `secret.yaml` - it already contains the encoded password.

    ```bash
    cat secret.yaml
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
    kubectl create -f secret.yaml
    ```

    ```
    secret/lfsecret created
    ```

4. Edit `second.yaml` to mount the secret as a volume at `/mysqlpassword`.

    ```bash
    vim second.yaml
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

5. Delete, recreate, and verify the secret is accessible inside the container.

    ```bash
    kubectl delete pod secondapp
    kubectl create -f second.yaml
    kubectl get pod secondapp
    ```

    ```
    NAME         READY   STATUS    RESTARTS   AGE
    secondapp    1/1     Running   0          34s
    ```

    ```bash
    kubectl exec -ti secondapp -- /bin/sh
    ```

    Inside:

    ```
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

    The mount point uses symlinks - the actual data is in a timestamped directory, and `..data` is a symlink to it. This allows Kubernetes to atomically update secrets.

---

## Exercise 6.3: Working with ServiceAccounts

ServiceAccounts provide an identity for pod processes to interact with the Kubernetes API.

1. Return to the home directory and view existing secrets.

    ```bash
    cd
    kubectl get secrets
    kubectl get secrets --all-namespaces
    ```

2. Create the ServiceAccount from `serviceaccount.yaml`.

    ```bash
    kubectl create -f ~/app2/serviceaccount.yaml
    ```

    ```
    serviceaccount/secret-access-sa created
    ```

    ```bash
    kubectl get serviceaccounts
    ```

    ```
    NAME               SECRETS   AGE
    default            0         ...
    secret-access-sa   0         34s
    ```

3. View existing ClusterRoles and compare `admin` vs `cluster-admin`.

    ```bash
    kubectl get clusterroles
    kubectl get clusterroles admin -o yaml
    kubectl get clusterroles cluster-admin -o yaml
    ```

    Note that `cluster-admin` uses `*` (wildcard) for all resources and verbs.

4. Create the `secret-access-cr` ClusterRole from `clusterrole.yaml`. It grants only `get` and `list` on secrets.

    ```bash
    kubectl create -f ~/app2/clusterrole.yaml
    ```

    ```
    clusterrole.rbac.authorization.k8s.io/secret-access-cr created
    ```

    ```bash
    kubectl get clusterrole secret-access-cr -o yaml
    ```

5. Bind the ClusterRole to the ServiceAccount using `rolebinding.yaml`.

    ```bash
    kubectl create -f ~/app2/rolebinding.yaml
    ```

    ```
    rolebinding.rbac.authorization.k8s.io/secret-rb created
    ```

    ```bash
    kubectl get rolebindings
    ```

    ```
    NAME        AGE
    secret-rb   17s
    ```

6. Verify the current `secondapp` pod uses the `default` serviceAccount.

    ```bash
    kubectl get pod secondapp -o yaml | grep serviceAccount
    ```

    ```
    serviceAccount: default
    serviceAccountName: default
    ```

7. Edit `~/app2/second.yaml` and add `serviceAccountName: secret-access-sa` to the pod spec.

    ```bash
    vim ~/app2/second.yaml
    ```

    ```yaml
    spec:
      serviceAccountName: secret-access-sa   # <-- add this line
      securityContext:
        runAsUser: 1000
    ```

8. Delete and recreate. Verify the new serviceAccount is in use.

    ```bash
    kubectl delete pod secondapp
    kubectl create -f ~/app2/second.yaml
    kubectl get pod secondapp -o yaml | grep serviceAccount
    ```

    ```
    serviceAccount: secret-access-sa
    serviceAccountName: secret-access-sa
    ```

---

## Exercise 6.4: Implement a NetworkPolicy

!!! note "Calico enforces NetworkPolicies"
    This cluster uses Calico, which fully enforces NetworkPolicies. Once you apply the policy in the next exercise, traffic will be blocked as described.

1. Review `allclosed.yaml`. This deny-all policy matches every pod and blocks both ingress and egress.

    ```bash
    cat ~/app2/allclosed.yaml
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

2. Delete the existing `secondapp` pod and rebuild it with a `webserver` nginx container and a label.

    ```bash
    kubectl delete pod secondapp
    vim second.yaml
    ```

    ```yaml
    spec:
      serviceAccountName: secret-access-sa
    # securityContext:        # <-- comment out
    #   runAsUser: 1000       # <-- comment out
      containers:
      - name: webserver
        image: nginx          # <-- add this
      - name: busy
        image: busybox
        command:
    ```

    Also add `labels:` in the metadata section:

    ```yaml
    metadata:
      name: secondapp
      labels:
        example: second
    ```

    ```bash
    kubectl create -f second.yaml
    kubectl get pods
    ```

    ```
    NAME        READY   STATUS    RESTARTS   AGE
    secondapp   2/2     Running   0          5s
    ```

3. Create the NodePort service and set it to port 32000.

    ```bash
    kubectl create service nodeport secondapp --tcp=80
    ```

    ```
    service/secondapp created
    ```

    Edit the service to change the selector to `example: second` and set the nodePort to `32000`.

    ```bash
    kubectl edit svc secondapp
    ```

4. Verify the service and test access.

    ```bash
    kubectl get svc secondapp
    ```

    ```
    NAME        TYPE       CLUSTER-IP      PORT(S)
    secondapp   NodePort   10.97.96.75     80:32000/TCP
    ```

    ```bash
    curl http://10.97.96.75
    ```

5. Test egress from within the container.

    ```bash
    kubectl exec -it -c busy secondapp -- sh
    ```

    Inside:

    ```
    / $ nc -vz 127.0.0.1 80
    127.0.0.1 (127.0.0.1:80) open

    / $ nc -vz www.linux.com 80
    www.linux.com (151.101.185.5:80) open

    / $ exit
    ```

---

## Exercise 6.5: Testing the NetworkPolicy

!!! warning "Traffic will be blocked"
    With Calico, the policy below will be enforced. Expect curl and nc tests to time out once the deny-all policy is applied.

1. Apply the deny-all policy.

    ```bash
    kubectl create -f ~/app2/allclosed.yaml
    ```

    ```
    networkpolicy.networking.k8s.io/deny-default created
    ```

2. Test that traffic is now blocked.

    ```bash
    curl http://10.97.96.75
    kubectl exec -it -c busy secondapp -- sh
    ```

    ```
    / $ nc -vz www.linux.com 80
    / $ exit
    ```

3. Update the policy to remove the Egress restriction and replace.

    ```bash
    vim ~/app2/allclosed.yaml
    ```

    ```yaml
    spec:
      podSelector: {}
      policyTypes:
      - Ingress
    # - Egress    # <-- comment out
    ```

    ```bash
    kubectl replace -f ~/app2/allclosed.yaml
    ```

    ```
    networkpolicy.networking.k8s.io/deny-default replaced
    ```

4. Get the pod IP, then add an ingress rule to allow traffic from any pod.

    ```bash
    kubectl get pod secondapp -o wide
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
    kubectl replace -f ~/app2/allclosed.yaml
    ```

5. Test pod-to-pod ingress with ping using a temporary alpine pod.

    ```bash
    kubectl run -it test --rm=true --image alpine \
    ```

    ```
    -- ping -c5 <secondapp-pod-ip>
    ```

6. Update the policy to only allow TCP port 80, blocking ICMP.

    ```bash
    vim ~/app2/allclosed.yaml
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
    kubectl replace -f ~/app2/allclosed.yaml
    ```

    ```
    # HTTP allowed:
    ```

    ```bash
    curl http://10.97.96.75
    ```

    ```
    # ICMP blocked:
    ```

    ```bash
    kubectl run -it test --rm=true --image alpine \
    ```

    ```
    -- ping -c5 <secondapp-pod-ip>
    ```

7. Delete the NetworkPolicy so the registry and other pods remain accessible for future chapters.

    ```bash
    kubectl delete networkpolicies deny-default
    ```

    ```
    networkpolicy.networking.k8s.io "deny-default" deleted
    ```

---

## Exercise 6.6: Domain Review

!!! important
    Source pages and content in this review can change at any time. Always check current information before your exam.

Revisit the CKAD curriculum for topics covered in this chapter:

- Understand authentication, authorization, and admission control
- Create and consume Secrets
- Understand ServiceAccounts
- Understand SecurityContexts
- Demonstrate basic understanding of NetworkPolicies

- Create a new deployment using the `nginx` image.
- Create a `LoadBalancer` service to expose the deployment on port 80.
- Create a new NetworkPolicy called `netblock` that blocks all traffic to pods in this deployment only. Verify the object is created.
- Update `netblock` to allow ingress traffic on port 80 only. Verify with curl.
- Create a pod using `security-review1.yaml`.

    ```bash
    kubectl create -f ~/app2/security-review1.yaml
    ```

- Check the pod status.

    ```bash
    kubectl get pod securityreview
    kubectl describe pod securityreview
    kubectl logs securityreview
    ```

- Exec into the container (or use `kubectl debug`) to find the UID that nginx actually needs. Hint: check `/etc/passwd` inside the nginx image for the `nginx` user's UID.
- Edit `security-review1.yaml` and set the container `runAsUser` to the correct nginx UID. Recreate the pod and verify it runs.
- Create a new ServiceAccount called `securityaccount`.

    ```bash
    kubectl create serviceaccount securityaccount
    ```

- Create a ClusterRole named `secrole` that allows `create`, `delete`, and `list` of pods in all apiGroups.
- Bind `secrole` to `securityaccount`.
- Create a token for `securityaccount` and save only the token value to `/tmp/securitytoken`.

    ```bash
    kubectl create token securityaccount > /tmp/securitytoken
    ```

- Clean up all resources created during this review.

    ```bash
    kubectl delete deployment nginx --ignore-not-found
    kubectl delete svc nginx --ignore-not-found
    kubectl delete networkpolicy netblock --ignore-not-found
    kubectl delete pod securityreview --ignore-not-found
    kubectl delete serviceaccount securityaccount --ignore-not-found
    kubectl delete clusterrole secrole --ignore-not-found
    kubectl delete clusterrolebinding secrole --ignore-not-found
    ```
