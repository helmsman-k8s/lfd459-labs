# Chapter 7 ? Exposing Applications

## Lab Overview

In this chapter you will work with all four service types (ClusterIP, NodePort, LoadBalancer, ExternalName), explore CoreDNS service discovery across namespaces, deploy an NGINX ingress controller via Helm, configure ingress rules for multiple virtual hosts, and optionally install Linkerd as a service mesh.

!!! note "Prerequisites"
    This chapter continues from Chapter 6. The `secondapp` pod (with `webserver` nginx and `busy` busybox containers, label `example: second`) should be running. If not, recreate it from `~/app2/second.yaml`.

---

## Exercise 7.1: Expose a Service

### ClusterIP ? Internal Access Only

1. Connect to the **controller** node and move into the `app2` directory. View existing services.

    ```bash
    guru@controller:~$ cd ~/app2
    guru@controller:~/app2$ kubectl get svc
    NAME         TYPE        CLUSTER-IP       PORT(S)
    kubernetes   ClusterIP   10.96.0.1        443/TCP
    nginx        ClusterIP   10.108.95.67     443/TCP
    registry     ClusterIP   10.105.119.236   5000/TCP
    secondapp    NodePort    10.111.26.8      80:32000/TCP
    ```

2. Save the existing `secondapp` service definition and delete it.

    ```bash
    guru@controller:~/app2$ kubectl get svc secondapp -o yaml > oldservice.yaml
    guru@controller:~/app2$ kubectl delete svc secondapp
    service "secondapp" deleted
    ```

3. Recreate the service using `newservice.yaml`. Note that no `type` is specified ? it defaults to `ClusterIP`.

    ```bash
    guru@controller:~/app2$ kubectl create -f newservice.yaml
    service/secondapp created

    guru@controller:~/app2$ kubectl get svc secondapp
    NAME        TYPE        CLUSTER-IP      PORT(S)   AGE
    secondapp   ClusterIP   10.98.148.52    80/TCP    14s
    ```

4. Test access to the ClusterIP. You should see the nginx welcome page.

    ```bash
    guru@controller:~/app2$ curl http://10.98.148.52
    ```

### Service Update Pattern

5. Create a second web server using the `httpd` image. Its default page differs from nginx.

    ```bash
    guru@controller:~/app2$ kubectl create deployment newserver --image=httpd
    deployment.apps/newserver created
    ```

6. Find the label that `newserver`'s pods use.

    ```bash
    guru@controller:~/app2$ kubectl get deployment newserver -o wide
    ```

    Note the `SELECTOR` column ? it will be `app=newserver`.

7. Edit the `secondapp` service to switch the selector to `newserver`'s label.

    ```bash
    guru@controller:~/app2$ kubectl edit svc secondapp
    ```

    Change `selector.example: second` to `selector.app: newserver`.

8. Test ? traffic should now show the httpd default page instead of nginx.

    ```bash
    guru@controller:~/app2$ curl http://10.98.148.52
    <html><body><h1>It works!</h1></body></html>
    ```

9. Edit the selector back to `example: second` and test again ? nginx returns. Then delete the `newserver` deployment.

    ```bash
    guru@controller:~/app2$ kubectl delete deployment newserver
    ```

### NodePort ? External Access

10. Edit `newservice.yaml` to add a NodePort type and pin the high port to `32000`.

    ```bash
    guru@controller:~/app2$ vim newservice.yaml
    ```

    Add the `nodePort` and `type` fields inside `ports`:

    ```yaml
    spec:
      ports:
      - port: 80
        protocol: TCP
        nodePort: 32000    # <-- add
      type: NodePort        # <-- add
      selector:
        example: second
    ```

11. Delete and recreate the service. Note a new ClusterIP is assigned.

    ```bash
    guru@controller:~/app2$ kubectl delete svc secondapp
    guru@controller:~/app2$ kubectl create -f newservice.yaml
    guru@controller:~/app2$ kubectl get svc secondapp
    NAME        TYPE       CLUSTER-IP       PORT(S)
    secondapp   NodePort   10.109.134.221   80:32000/TCP
    ```

12. Test via the ClusterIP (internal) and via any node IP on port 32000 (external).

    ```bash
    guru@controller:~/app2$ curl http://10.109.134.221
    guru@controller:~/app2$ curl http://<worker1-ip>:32000
    guru@controller:~/app2$ curl http://<worker2-ip>:32000
    ```

### LoadBalancer

13. Edit `newservice.yaml` and change `type: NodePort` to `type: LoadBalancer`.

    ```bash
    guru@controller:~/app2$ vim newservice.yaml
    ```

    ```yaml
    type: LoadBalancer    # <-- change from NodePort
    ```

    ```bash
    guru@controller:~/app2$ kubectl delete svc secondapp
    guru@controller:~/app2$ kubectl create -f newservice.yaml
    ```

14. Check the service. `EXTERNAL-IP` will remain `<pending>` ? there is no cloud load balancer in this environment. The NodePort still works.

    ```bash
    guru@controller:~/app2$ kubectl get svc secondapp
    NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
    secondapp   LoadBalancer   10.109.26.21    <pending>     80:32000/TCP
    ```

    Access still works via the NodePort.

    ```bash
    guru@controller:~/app2$ curl http://<worker1-ip>:32000
    ```

### CoreDNS Service Discovery

15. Log into the `busy` container of `secondapp` and explore DNS resolution.

    ```bash
    guru@controller:~/app2$ kubectl exec -it secondapp -c busy -- sh
    ```

    Inside the container:

    ```sh
    # Resolve the secondapp service ? only works for services in the same namespace
    / $ nslookup secondapp
    Name: secondapp.default.svc.cluster.local
    Address: 10.96.214.133

    # Resolve the registry service
    / $ nslookup registry
    Name: registry.default.svc.cluster.local
    Address: 10.110.95.21

    # Reverse-lookup the DNS server IP
    / $ nslookup 10.96.0.10
    10.0.96.10.in-addr.arpa  name = kube-dns.kube-system.svc.cluster.local

    # Short name search fails (not in default namespace)
    / $ nslookup kube-dns
    ** server can't find kube-dns.default.svc.cluster.local: NXDOMAIN

    # FQDN works
    / $ nslookup kube-dns.kube-system.svc.cluster.local
    Name: kube-dns.kube-system.svc.cluster.local
    Address: 10.96.0.10

    / $ exit
    ```

    !!! note
        CoreDNS appends `.default.svc.cluster.local` when resolving short names. Services in other namespaces require the FQDN: `<service>.<namespace>.svc.cluster.local`.

### Cross-Namespace DNS

16. Create a new namespace `multitenant` with a deployment and service.

    ```bash
    guru@controller:~/app2$ kubectl create ns multitenant
    guru@controller:~/app2$ kubectl -n multitenant create deployment mainapp --image=nginx
    guru@controller:~/app2$ kubectl -n multitenant expose deployment mainapp \
    --name=shopping --type=NodePort --port=80
    ```

17. Log back into the `busy` container and test cross-namespace DNS.

    ```bash
    guru@controller:~/app2$ kubectl exec -it secondapp -c busy -- sh
    ```

    Inside:

    ```sh
    # Short name fails ? shopping is not in the default namespace
    / $ nslookup shopping
    ** server can't find shopping.default.svc.cluster.local: NXDOMAIN

    # FQDN resolves correctly
    / $ nslookup shopping.multitenant.svc.cluster.local
    Name: shopping.multitenant.svc.cluster.local
    Address: 10.101.4.142

    # Partial FQDN with namespace works for wget
    / $ wget -O - shopping.multitenant
    <!DOCTYPE html>
    <html>
    <head><title>Welcome to nginx!</title>
    ...

    / $ exit
    ```

---

## Exercise 7.2: Service Mesh and Ingress Controller

### NGINX Ingress Controller via Helm

An ingress controller allows you to route traffic to multiple services using a single entry point based on HTTP host headers or paths.

1. Make sure Helm is available.

    ```bash
    guru@controller:~$ helm version
    ```

    If not found, install it:

    ```bash
    guru@controller:~$ wget https://get.helm.sh/helm-v3.18.1-linux-amd64.tar.gz
    guru@controller:~$ tar -xf helm-v3.18.1-linux-amd64.tar.gz
    guru@controller:~$ sudo cp linux-amd64/helm /usr/local/bin/
    ```

2. Add the NGINX ingress Helm repository and update.

    ```bash
    guru@controller:~$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    guru@controller:~$ helm repo update
    ```

3. Download the chart and edit `values.yaml` to use a `DaemonSet` instead of a `Deployment` (one ingress pod per node).

    ```bash
    guru@controller:~$ helm fetch ingress-nginx/ingress-nginx --untar
    guru@controller:~$ cd ingress-nginx
    guru@controller:~/ingress-nginx$ vim values.yaml
    ```

    Find the `kind:` line around line 150 and change it:

    ```yaml
    ## DaemonSet or Deployment
    kind: DaemonSet    # <-- change from Deployment
    ```

4. Install the ingress controller using the local chart.

    ```bash
    guru@controller:~/ingress-nginx$ helm install myingress .
    NAME: myingress
    STATUS: deployed
    ```

5. Verify the controller pods are running ? one per node (DaemonSet).

    ```bash
    guru@controller:~/ingress-nginx$ kubectl get pods --all-namespaces | grep myingress
    default   myingress-ingress-nginx-controller-xxxxx   1/1   Running   0   20s   controller
    default   myingress-ingress-nginx-controller-yyyyy   1/1   Running   0   20s   worker1
    default   myingress-ingress-nginx-controller-zzzzz   1/1   Running   0   20s   worker2
    ```

6. Check the ingress controller service. `EXTERNAL-IP` will be `<pending>` ? that's expected.

    ```bash
    guru@controller:~/ingress-nginx$ kubectl get svc | grep myingress
    myingress-ingress-nginx-controller   LoadBalancer   10.104.227.79   <pending>   80:32558/TCP,443:30219/TCP
    ```

7. Note the pod IPs of the ingress controller pods (these are the IPs you'll use directly for testing).

    ```bash
    guru@controller:~/ingress-nginx$ kubectl get pod -o wide | grep myingress
    ```

### Creating an Ingress Rule

8. Return to the home directory and review `ingress.yaml`. It routes traffic for `Host: www.example.com` to the `secondapp` service.

    ```bash
    guru@controller:~/ingress-nginx$ cd ~/app2
    guru@controller:~/app2$ cat ~/lfd459/ch07-exposing-apps/ingress.yaml
    ```

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-test
      annotations:
        nginx.ingress.kubernetes.io/service-upstream: "true"
      namespace: default
    spec:
      ingressClassName: nginx
      rules:
      - host: www.example.com
        http:
          paths:
          - backend:
              service:
                name: secondapp
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
    ```

    !!! warning "Old ingress file"
        A file called `ingress_rule.yaml` exists in the lab files ? it uses `apiVersion: networking.k8s.io/v1beta1` which was **removed in Kubernetes 1.22**. Do not use it. Always use `ingress.yaml` (v1).

9. Copy and apply the ingress rule.

    ```bash
    guru@controller:~/app2$ cp ~/lfd459/ch07-exposing-apps/ingress.yaml .
    guru@controller:~/app2$ kubectl create -f ingress.yaml
    ingress.networking.k8s.io/ingress-test created

    guru@controller:~/app2$ kubectl get ingress
    NAME           CLASS   HOSTS             ADDRESS   PORTS   AGE
    ingress-test   nginx   www.example.com             80      5s
    ```

10. Test the ingress. Without the correct `Host` header you get a 404. With the header, nginx responds.

    ```bash
    # Get the ingress controller pod IP on the controller node
    guru@controller:~/app2$ INGRESS_IP=$(kubectl get pods -o wide | \
    grep myingress | grep controller | awk 'NR==1{print $6}')
    echo $INGRESS_IP

    # Without Host header ? 404 (ingress has no default backend)
    guru@controller:~/app2$ curl $INGRESS_IP
    <html><head><title>404 Not Found</title></head>...</html>

    # With matching Host header ? nginx welcome page
    guru@controller:~/app2$ curl -H "Host: www.example.com" http://$INGRESS_IP
    <!DOCTYPE html>
    <html><head><title>Welcome to nginx!</title>...
    ```

    !!! note "Using ClusterIP instead"
        You can also use the ingress controller's ClusterIP with the Host header:
        ```bash
        curl -H "Host: www.example.com" http://10.104.227.79
        ```

### Adding a Second Virtual Host

11. Deploy a third web server and customise its default page.

    ```bash
    guru@controller:~/app2$ kubectl create deployment thirdpage --image=nginx
    guru@controller:~/app2$ kubectl expose deployment thirdpage --port=80 --type=NodePort
    ```

12. Label the pod so it can be targeted (use Tab completion for the pod name).

    ```bash
    guru@controller:~/app2$ kubectl label pod thirdpage-<Tab> example=third
    ```

13. Exec into the pod and edit the nginx default page to say "Third Page".

    ```bash
    guru@controller:~/app2$ kubectl exec -it thirdpage-<Tab> -- /bin/bash
    ```

    Inside:

    ```bash
    root@thirdpage:/# apt-get update -qq && apt-get install vim -y -qq
    root@thirdpage:/# vim /usr/share/nginx/html/index.html
    ```

    Change `<title>Welcome to nginx!</title>` to `<title>Third Page</title>`, then exit.

14. Edit the ingress resource to add a second host rule for `thirdpage.org`.

    ```bash
    guru@controller:~/app2$ kubectl edit ingress ingress-test
    ```

    Add a second rule block under `spec.rules`:

    ```yaml
    spec:
      rules:
      - host: thirdpage.org
        http:
          paths:
          - backend:
              service:
                name: thirdpage
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
      - host: www.example.com
        http:
          paths:
          - backend:
              service:
                name: secondapp
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
    ```

    !!! tip
        The `ingress-after.yaml` reference file in `~/lfd459/ch07-exposing-apps/` shows the final ingress YAML with both rules.

15. Test both virtual hosts.

    ```bash
    # Should show "Third Page"
    guru@controller:~/app2$ curl -H "Host: thirdpage.org" http://$INGRESS_IP
    <!DOCTYPE html><html><head><title>Third Page</title>...

    # Should still show nginx default
    guru@controller:~/app2$ curl -H "Host: www.example.com" http://$INGRESS_IP
    <!DOCTYPE html><html><head><title>Welcome to nginx!</title>...
    ```

16. Consider: how would you switch `thirdpage.org` traffic to an `httpd` server by editing only the ingress rule ? without touching the pods or services?

---

## Exercise 7.2b: Linkerd Service Mesh (Optional)

!!! note "Optional exploration"
    Linkerd is not tested on the CKAD exam. This exercise is provided for exploration. It takes 10?20 minutes and requires internet access. Skip it if you are focused on exam preparation.

The `setupLinkerd.txt` file in `~/lfd459/ch07-exposing-apps/` contains the install commands. Key steps:

```bash
guru@controller:~$ curl -sL run.linkerd.io/install | sh
guru@controller:~$ export PATH=$PATH:/home/guru/.linkerd2/bin
guru@controller:~$ echo 'export PATH=$PATH:/home/guru/.linkerd2/bin' >> ~/.bashrc

# Pre-flight check
guru@controller:~$ linkerd check --pre

# Install (use --crds separately for newer versions)
guru@controller:~$ linkerd install --crds | kubectl apply -f -
guru@controller:~$ linkerd install | kubectl apply -f -
guru@controller:~$ linkerd check

# Install the viz dashboard
guru@controller:~$ linkerd viz install | kubectl apply -f -
guru@controller:~$ linkerd viz check
guru@controller:~$ linkerd viz dashboard &
```

To expose the dashboard externally, edit the `web` deployment to remove the enforced-host restriction, and change the `web` service in `linkerd-viz` namespace to `NodePort` on port `31500`.

To inject Linkerd into a deployment:

```bash
guru@controller:~$ kubectl -n multitenant get deploy mainapp -o yaml | \
linkerd inject - | kubectl apply -f -
```

To clean up when done:

```bash
guru@controller:~$ linkerd viz uninstall | kubectl delete -f -
guru@controller:~$ linkerd uninstall | kubectl delete -f -
```

---

## Exercise 7.3: Domain Review

!!! warning "Important"
    Source pages and content in this review can change at any time. Always check current information before your exam.

Revisit the CKAD curriculum for topics covered in this chapter:

- Use Kubernetes primitives to implement common deployment strategies (blue/green or canary)
- Use the Helm package manager to deploy existing packages
- Provide and troubleshoot access to applications via services
- Use Ingress rules to expose applications

1. Create a new pod called `webone` running nginx. Expose port 80.

2. Create a service named `webone-svc` accessible from outside the cluster.

3. Update both pod and service with matching selectors so traffic reaches the webserver.

4. Change the service type so it is only accessible from within the cluster. Verify external access no longer works but internal curl does.

5. Deploy another pod called `webtwo` running the `wlniao/website` image. Create a service `webtwo-svc` accessible only from within the cluster. Verify the default pages for each server are different.

6. Install and configure an ingress controller so that requests for `webone.com` show the nginx default page and requests for `webtwo.org` show the `wlniao/website` default page.

7. Clean up all resources created in this review.

    ```bash
    guru@controller:~$ kubectl delete pod webone webtwo --ignore-not-found
    guru@controller:~$ kubectl delete svc webone-svc webtwo-svc --ignore-not-found
    guru@controller:~$ kubectl delete ingress --all --ignore-not-found
    ```
