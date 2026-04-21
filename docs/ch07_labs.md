# Chapter 7 - Exposing Applications

## Lab Overview

In this chapter you will work with all four service types (ClusterIP, NodePort, LoadBalancer, ExternalName), explore CoreDNS service discovery across namespaces, deploy an NGINX ingress controller via Helm, configure ingress rules for multiple virtual hosts, and optionally install Linkerd as a service mesh.

!!! note "Prerequisites"
    This chapter continues from Chapter 6. The `secondapp` pod (with `webserver` nginx and `busy` busybox containers, label `example: second`) should be running. If not, recreate it from `~/app2/second.yaml`.

---

## Exercise 7.1: Expose a Service

### ClusterIP - Internal Access Only

1. Connect to the **controller** node and move into the chapter lab directory.

    ```bash
    cd ~/lfd459/ch07-exposing-apps
    kubectl get svc
    ```
    # output

    ```
    NAME         TYPE        CLUSTER-IP       PORT(S)
    kubernetes   ClusterIP   10.96.0.1        443/TCP
    nginx        ClusterIP   10.108.95.67     443/TCP
    registry     ClusterIP   10.105.119.236   5000/TCP
    secondapp    NodePort    10.111.26.8      80:32000/TCP
    ```
    # output

2. Save the existing `secondapp` service definition and delete it.

    ```bash
    kubectl get svc secondapp -o yaml > oldservice.yaml
    kubectl delete svc secondapp
    ```

    ```
    # output
    service "secondapp" deleted
    ```

3. Recreate the service using `newservice.yaml`. Note that no `type` is specified - it defaults to `ClusterIP`.

    ```bash
    kubectl create -f newservice.yaml
    ```
    # output

    ```
    service/secondapp created
    ```
    # output

    ```bash
    kubectl get svc secondapp
    ```

    ```
    # output
    NAME        TYPE        CLUSTER-IP      PORT(S)   AGE
    secondapp   ClusterIP   10.98.148.52    80/TCP    14s
    ```

4. Test access to the ClusterIP. You should see the nginx welcome page.

    ```bash
    curl http://10.98.148.52
    ```
    # output

### Service Update Pattern

1. Create a second web server using the `httpd` image. Its default page differs from nginx.

    ```bash
    kubectl create deployment newserver --image=httpd
    ```

    ```
    # output
    deployment.apps/newserver created
    ```

2. Find the label that `newserver`'s pods use.

    ```bash
    kubectl get deployment newserver -o wide
    ```
    # output

    Note the `SELECTOR` column - it will be `app=newserver`.

3. Edit the `secondapp` service to switch the selector to `newserver`'s label.

    ```bash
    kubectl edit svc secondapp
    ```

    Change `selector.example: second` to `selector.app: newserver`.

4. Test - traffic should now show the httpd default page instead of nginx.

    ```bash
    curl http://10.98.148.52
    ```
    # output

    ```
    <html><body><h1>It works!</h1></body></html>
    ```
    # output

5. Edit the selector back to `example: second` and test again - nginx returns. Then delete the `newserver` deployment.

    ```bash
    kubectl delete deployment newserver
    ```

### NodePort - External Access

1. Edit `newservice.yaml` to add a NodePort type and pin the high port to `32000`.

    ```bash
    vim newservice.yaml
    ```
    # output

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

    !!! tip
        Pre-built version available: `kubectl create -f newservice-v2.yaml`

2. Delete and recreate the service. Note a new ClusterIP is assigned.

    ```bash
    kubectl delete svc secondapp
    kubectl create -f newservice.yaml
    kubectl get svc secondapp
    ```
    # output

    ```
    NAME        TYPE       CLUSTER-IP       PORT(S)
    secondapp   NodePort   10.109.134.221   80:32000/TCP
    ```
    # output

3. Test via the ClusterIP (internal) and via any node IP on port 32000 (external).

    ```bash
    curl http://10.109.134.221
    curl http://worker1:32000
    curl http://worker2:32000
    ```

### LoadBalancer

1. Edit `newservice.yaml` and change `type: NodePort` to `type: LoadBalancer`.

    ```bash
    vim newservice.yaml
    ```
    # output

    ```yaml
    type: LoadBalancer    # <-- change from NodePort
    ```

    !!! tip
        Pre-built version available: `kubectl create -f newservice-v3.yaml`

    ```bash
    kubectl delete svc secondapp
    kubectl create -f newservice.yaml
    ```
    # output

2. Check the service. `EXTERNAL-IP` will remain `<pending>` - there is no cloud load balancer in this environment. The NodePort still works.

    ```bash
    kubectl get svc secondapp
    ```

    ```
    # output
    NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
    secondapp   LoadBalancer   10.109.26.21    <pending>     80:32000/TCP
    ```

    Access still works via the NodePort.

    ```bash
    curl http://worker1:32000
    ```
    # output

### CoreDNS Service Discovery

1. Log into the `busy` container of `secondapp` and explore DNS resolution.

    ```bash
    kubectl exec -it secondapp -c busy -- sh
    ```

    Inside the container:

    ```bash
    # Resolve the secondapp service - only works for services in the same namespace
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
    # output

    !!! note
        CoreDNS appends `.default.svc.cluster.local` when resolving short names. Services in other namespaces require the FQDN: `<service>.<namespace>.svc.cluster.local`.

### Cross-Namespace DNS

1. Create a new namespace `multitenant` with a deployment and service.

    ```bash
    kubectl create ns multitenant
    kubectl -n multitenant create deployment mainapp --image=nginx
    kubectl -n multitenant expose deployment mainapp \
    ```

    ```
    # output
    --name=shopping --type=NodePort --port=80
    ```

2. Log back into the `busy` container and test cross-namespace DNS.

    ```bash
    kubectl exec -it secondapp -c busy -- sh
    ```
    # output

    Inside:

    ```bash
    # Short name fails - shopping is not in the default namespace
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
    helm version
    ```
    # output

    If not found, install it:

    !!! tip "Helm version"
        Check the latest stable release at <https://github.com/helm/helm/releases> before downloading. Replace `v3.18.1` below with the current version if it differs.

    ```bash
    wget https://get.helm.sh/helm-v3.18.1-linux-amd64.tar.gz
    tar -xf helm-v3.18.1-linux-amd64.tar.gz
    sudo cp linux-amd64/helm /usr/local/bin/
    ```

2. Add the NGINX ingress Helm repository and update.

    ```bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    ```
    # output

3. Download the chart and edit `values.yaml` to use a `DaemonSet` instead of a `Deployment` (one ingress pod per node).

    ```bash
    helm fetch ingress-nginx/ingress-nginx --untar
    cd ingress-nginx
    vim values.yaml
    ```

    Find the `kind:` line around line 150 and change it:

    ```yaml
    ## DaemonSet or Deployment
    kind: DaemonSet    # <-- change from Deployment
    ```
    # output

4. Install the ingress controller using the local chart.

    ```bash
    helm install myingress .
    ```

    ```
    # output
    NAME: myingress
    STATUS: deployed
    ```

5. Verify the controller pods are running - one per node (DaemonSet).

    ```bash
    kubectl get pods --all-namespaces | grep myingress
    ```
    # output

    ```
    default   myingress-ingress-nginx-controller-xxxxx   1/1   Running   0   20s   controller
    default   myingress-ingress-nginx-controller-yyyyy   1/1   Running   0   20s   worker1
    default   myingress-ingress-nginx-controller-zzzzz   1/1   Running   0   20s   worker2
    ```
    # output

6. Check the ingress controller service. `EXTERNAL-IP` will be `<pending>` - that's expected.

    ```bash
    kubectl get svc | grep myingress
    ```

    ```
    # output
    myingress-ingress-nginx-controller   LoadBalancer   10.104.227.79   <pending>   80:32558/TCP,443:30219/TCP
    ```

7. Note the pod IPs of the ingress controller pods (these are the IPs you'll use directly for testing).

    ```bash
    kubectl get pod -o wide | grep myingress
    ```
    # output

### Creating an Ingress Rule

1. Review `ingress.yaml` in the chapter directory. It routes traffic for `Host: www.example.com` to the `secondapp` service.

    ```bash
    cat ingress.yaml
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
    # output

    !!! warning "Old ingress file"
        A file called `ingress_rule.yaml` exists in the lab files - it uses `apiVersion: networking.k8s.io/v1beta1` which was **removed in Kubernetes 1.22**. Do not use it. Always use `ingress.yaml` (v1).

2. Copy and apply the ingress rule.

    ```bash
    kubectl create -f ~/lfd459/ch07-exposing-apps/ingress.yaml
    ```

    ```
    # output
    ingress.networking.k8s.io/ingress-test created
    ```

    ```bash
    kubectl get ingress
    ```
    # output

    ```
    NAME           CLASS   HOSTS             ADDRESS   PORTS   AGE
    ingress-test   nginx   www.example.com             80      5s
    ```
    # output

3. Test the ingress. Without the correct `Host` header you get a 404. With the header, nginx responds.

    ```
    # Get the ingress controller pod IP on the controller node
    INGRESS_IP=$(kubectl get pods -o wide | \
    grep myingress | grep controller | awk 'NR==1{print $6}')
    ```
    # output

    ```bash
    echo $INGRESS_IP
    ```

    ```
    # output
    # Without Host header - 404 (ingress has no default backend)
    ```

    ```bash
    curl $INGRESS_IP
    ```
    # output

    ```
    <html><head><title>404 Not Found</title></head>...</html>

    # With matching Host header - nginx welcome page
    ```
    # output

    ```bash
    curl -H "Host: www.example.com" http://$INGRESS_IP
    ```

    ```
    # output
    <!DOCTYPE html>
    <html><head><title>Welcome to nginx!</title>...
    ```

    !!! tip "Using ClusterIP instead"
        You can also use the ingress controller's ClusterIP with the Host header:
        ```bash
        curl -H "Host: www.example.com" http://10.104.227.79
        ```

### Adding a Second Virtual Host

1. Deploy a third web server and customise its default page.

    ```bash
        ```

        ```bash
    kubectl create deployment thirdpage --image=nginx
    kubectl expose deployment thirdpage --port=80 --type=NodePort
        ```

2. Label the pod so it can be targeted (use Tab completion for the pod name).

    ```bash
    kubectl label pod thirdpage-<Tab> example=third
    ```
    # output

3. Exec into the pod and edit the nginx default page to say "Third Page".

    ```bash
    kubectl exec -it thirdpage-<Tab> -- /bin/bash
    ```

    Inside:

    ```bash
    root@thirdpage:/# apt-get update -qq && apt-get install vim -y -qq
    root@thirdpage:/# vim /usr/share/nginx/html/index.html
    ```
    # output

    Change `<title>Welcome to nginx!</title>` to `<title>Third Page</title>`, then exit.

4. Edit the ingress resource to add a second host rule for `thirdpage.org`.

    ```bash
    kubectl edit ingress ingress-test
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
    # output

    !!! tip
        Pre-built version available: `kubectl replace -f ingress-v2.yaml`

5. Test both virtual hosts.

    ```bash
    # Should show "Third Page"
    ```

    ```bash
    curl -H "Host: thirdpage.org" http://$INGRESS_IP
    ```
    # output

    ```
    <!DOCTYPE html><html><head><title>Third Page</title>...

    # Should still show nginx default
    ```
    # output

    ```bash
    curl -H "Host: www.example.com" http://$INGRESS_IP
    ```

    ```
    # output
    <!DOCTYPE html><html><head><title>Welcome to nginx!</title>...
    ```

6. Consider: how would you switch `thirdpage.org` traffic to an `httpd` server by editing only the ingress rule - without touching the pods or services?

---

## Service Mesh

A **service mesh** is an infrastructure layer that handles service-to-service communication inside a cluster. Rather than building resilience logic (retries, timeouts, mutual TLS, circuit breaking) into each application, a service mesh offloads it to a sidecar proxy injected alongside every pod.

Key capabilities a service mesh provides:

- **Mutual TLS (mTLS)** - encrypts and authenticates all traffic between services automatically
- **Observability** - metrics, traces, and traffic topology without application changes
- **Traffic management** - canary releases, traffic splitting, retries, timeouts at the network level
- **Circuit breaking** - automatically stop sending traffic to failing services

Popular implementations include **Linkerd**, **Istio**, and **Cilium Service Mesh**. They all follow the same pattern: a control plane that manages configuration, and data plane sidecar proxies (typically Envoy or a lightweight alternative) that intercept pod traffic.

!!! note "CKAD exam scope"
    The CKAD exam does not test service mesh configuration. You are expected to understand what a service mesh is and the problem it solves, not how to install or operate one.


---

## Exercise 7.3: Domain Review

!!! important
    Source pages and content in this review can change at any time. Always check current information before your exam.

Revisit the CKAD curriculum for topics covered in this chapter:

- Use Kubernetes primitives to implement common deployment strategies (blue/green or canary)
- Use the Helm package manager to deploy existing packages
- Provide and troubleshoot access to applications via services
- Use Ingress rules to expose applications

- Create a new pod called `webone` running nginx. Expose port 80.
- Create a service named `webone-svc` accessible from outside the cluster.
- Update both pod and service with matching selectors so traffic reaches the webserver.
- Change the service type so it is only accessible from within the cluster. Verify external access no longer works but internal curl does.
- Deploy another pod called `webtwo` running the `wlniao/website` image. Create a service `webtwo-svc` accessible only from within the cluster. Verify the default pages for each server are different.
- Install and configure an ingress controller so that requests for `webone.com` show the nginx default page and requests for `webtwo.org` show the `wlniao/website` default page.
- Clean up all resources created in this review.

    ```bash
    kubectl delete pod webone webtwo --ignore-not-found
    kubectl delete svc webone-svc webtwo-svc --ignore-not-found
    kubectl delete ingress --all --ignore-not-found
    ```
    # output
