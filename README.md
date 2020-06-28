# k8s-nginx-ingress-controller


# What is Ingress?

Traditionally, you would create a LoadBalancer service for each public system you want to expose. This can get rather expensive. Ingress gives you a way to route requests to services based on the request host or path, centralizing a number of services into a single entrypoint.

The detail documentation for deploying a nginx ingress controller on AWS is as follows:

https://kubernetes.github.io/ingress-nginx/deploy/#aws

In AWS we use an Elastic Load Balancer (ELB) to expose the NGINX Ingress controller behind a Service of Type=LoadBalancer. Since Kubernetes v1.9.0 it is possible to use a classic load balancer (ELB) or network load balancer (NLB) Please check the elastic load balancing AWS details page

ELASTIC LOAD BALANCER - ELB

This setup requires to choose in which layer (L4 or L7) we want to configure the ELB:

Layer 4: use TCP as the listener protocol for ports 80 and 443.

Layer 7: use HTTP as the listener protocol for port 80 and terminate TLS in the ELB


# Ingress Resources

Ingress is split up into two main pieces. The first is an Ingress resource, which defines how you want requests routed to the backing services.
For example, let's deploy a hello-world service with 2 Pods running in your namespace and apply the hello-world ingress resource file as below:


```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-world
spec:
  rules:
  - http:
      paths:
      - path: /api/hello-world
        backend:
          serviceName: hello-world
          servicePort: 80
```

Let’s take a look when an Ingress resource is deployed, how does the ingress controller translate it into Nginx configuration?

For API path /api/hello-world, through an [upstream directive](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) as below, it will route incoming traffic to Service hello-world with 2 destination Pod IPs on container port 8080 in the namespace demo. Pretty straightforward, right? It is very similar to our iptables or ipvs routing table

```
# Custom code snippet configured in the configuration configmap

	upstream demo-hello-world-80 {
		least_conn;

		keepalive 32;

		server 10.0.87.97:8080 max_fails=0 fail_timeout=0;
		server 10.0.137.64:8080 max_fails=0 fail_timeout=0;
	}

location /api/hello-world {
			log_by_lua_block {
			}
			port_in_redirect off;

			set $proxy_upstream_name "demo-hello-world-80";

			set $namespace      "demo";
			set $ingress_name   "hello-world";
			set $service_name   "hello-world";
			...
			proxy_pass http://demo-hello-world-80;

			proxy_redirect                          off;

		}
```

So, Ingress on its own does not really do anything. You need something to listen to the Kubernetes API for Ingress resources and then handle requests that match them. This is where the second piece to the puzzle comes in — the Ingress Controller.

# Why do I need a load balancer in front of an ingress?

Ingress is tightly integrated into Kubernetes, meaning that your existing workflows around kubectl will likely extend nicely to managing ingress. An Ingress controller does not typically eliminate the need for an external load balancer , it simply adds an additional layer of routing and control behind the load balancer.

Pods and nodes are not guaranteed to live for the whole lifetime that the user intends: pods are ephemeral and vulnerable to kill signals from Kubernetes during occasions such as:

* Scaling.
* Memory or CPU saturation.
* Rescheduling for more efficient resource use.
* Downtime due to outside factors.


The load balancer (Kubernetes service) is a construct that stands as a single, fixed-service endpoint for a given set of pods or worker nodes. To take advantage of the previously-discussed benefits of a Load Balancer, we create a Kubernetes service of  type:loadbalancer with the different annotations, and this load balancer sits in front of the ingress controller – which is itself a pod or a set of pods. In AWS, for a set of EC2 compute instances managed by an Autoscaling Group, there should be a load balancer that acts as both a fixed referable address and a load balancing mechanism.


# Nginx Ingress relies on a Classic Load Balancer(ELB)
Nginx ingress controller can be deployed anywhere, and when initialized in AWS, it will create a classic ELB to expose the Nginx Ingress controller behind a Service of Type=LoadBalancer. This may be an issue for some people since ELB is considered a legacy technology and AWS is recommending to migrate existing ELB to Network Load Balancer(NLB). However, under regular traffic volume, it never becomes a problem for us.

If NLB is preferred in your cluster, the good news is: it is supported since v1.10.0 as an ALPHA feature as below. But we will be using ELB for our ingress controller

```
annotations:    
  # by default the type is elb
  service.beta.kubernetes.io/aws-load-balancer-type: elb
```

#Ingress rules

Each HTTP rule contains the following information:

* An optional host. In this example, no host is specified, so the rule applies to all inbound HTTP traffic through the IP address specified. If a host is provided (for example, foo.bar.com), the rules apply to that host.
* A list of paths (for example, /testpath), each of which has an associated backend defined with a serviceName and servicePort. Both the host and path must match the content of an incoming request before the load balancer directs traffic to the referenced Service.
* A backend is a combination of Service and port names as described in the Service doc. HTTP (and HTTPS) requests to the Ingress that matches the host and path of the rule are sent to the listed backend.

A default backend is often configured in an Ingress controller to service any requests that do not match a path in the spec


# Ingress Controller and Its Scope

By default, the Ingress Controller monitors for Ingress and other resources across all the namespaces. That is why a ClusterRole and a ClusterRoleBinding are required.
The Complete nginx_ingress_controller helm charts can be found in the following repo :

https://github.com/snigdhasambitak/k8s-nginx-ingress-controller

When getting started, people tend to create an ingress controller with default values and start to try things out, such as deploying the dashboard or migrating a few applications.

We took a close look at Nginx Ingress controller helm chart and it has the following settings:

```
controller.scope.enabled: default to false, watch all namespaces
controller.scope.namespace namespace to watch for ingress, default to empty

```
This means, by default, each Ingress controller will listen to all the ingress events from all the namespaces and add corresponding directives and rules into Nginx configuration file.

Let’s take another look at the ingress controller deployment as below. Notice when the chart is deployed, these settings are translated into a container argument called `--watch-namespace` .This might come in handy and save you some time during the debug process.

```
kubectl get deployments chart-1590035165-nginx-ingress-controller -o yaml

...
containers:
      - args:
        - /nginx-ingress-controller
        - --default-backend-service=k8s-ingress-nginx/chart-1590035165-nginx-ingress-default-backend
        - --publish-service=k8s-ingress-nginx/chart-1590035165-nginx-ingress-controller
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --configmap=k8s-ingress-nginx/chart-1590035165-nginx-ingress-controller
```

If you would like to only use the Ingress Controller for a certain namespace, you need to:

Set the `-watch-namespace` command line argument in the IC manifests.
for example,`-watch-namespace=my-namespace` https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/cli-arguments.md

## deploy ingress nginx with RBAC
If you want to deploy a nginx ingress controller with RBAC enabled you can use [ingress-controller-cluster-scope-with-rbac](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/example_configs/ingress-controller-cluster-scope-with-rbac.yaml)

## deploy ingress nginx without RBAC
If you want to deploy a nginx ingress controller without RBAC enabled you can use [ingress-controller-cluster-scope-without-rbac](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/example_configs/ingress-controller-namespace-scope-without-rbac.yaml)

# How to deploy the Nginx Ingress Controller

The nginx ingress controller can be deployed on any cloud provider `AWS`, `AZURE` and `Google Cloud`

## Create the cluster role and Bindings

* You should have the following cluster role and role bindings defined in your cluster in which you are trying to deploy the nginx ingress controller. The serviceAccountName associated with the containers in the ingress controller deployment must match the `nginx-ingress-serviceaccount` serviceAccount. The namespace references in the Deployment metadata, container arguments, and POD_NAMESPACE should be in the `k8s-ingress-nginx` namespace.

  * Cluster Role
  * Cluster Role Bindings
  * Service Account Used : nginx-ingress-serviceaccount
  * Namespace : k8s-ingress-nginx

## Create ingress nginx controller helm charts

* Once the cluster role and bindings are applied you need to create the helm charts that will deploy the nginx ingress controller in the `k8s-ingress-nginx` namespace. The helm charts can be found here [Nginx Ingress Controller Helm Charts](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller)

* Deploy the nginx ingress controller help charts from the base folder location of this project
  ```
  helm upgrade -i k8s .
  ```
* The important Configurations can be found below:

  # Source: [serviceaccount](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/templates/controller-serviceaccount.yaml)
  ```
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
    name: nginx-ingress-serviceaccount
    namespace: k8s-ingress-nginx
  ---
```
  # Source: [controller-configmap](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/templates/controller-configmap.yaml)

  ```apiVersion: v1
  kind: ConfigMap
  metadata:
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
    name: ingress-nginx-controller
    namespace: k8s-ingress-nginx
  data:
    # http-snippet: |
    #   server {
    #     listen 2443;
    #     return 308 https://$host$request_uri;
    #   }
    <!-- proxy-real-ip-cidr: 10.11.12.13/22 -->
    use-forwarded-headers: 'true'
    use-proxy-protocol: "true"
```
If `use-proxy-protocol` is enabled, proxy-real-ip-cidr defines the default the IP/network address of your external load balancer.


By default NGINX uses the content of the header X-Forwarded-For as the source of truth to get information about the client IP address. This works without issues in L7 if we configure the setting proxy-real-ip-cidr with the correct information of the IP/network address of trusted external load balancer.
If the ingress controller is running in AWS we need to use the VPC IPv4 CIDR
By using use-proxy-protocol Nginx uses the module ngx_http_realip_module reading the Forwarded IP and updating the remote_addr with the first IP (client IP). After this change, Nginx traces will displace client public IP instead of ELB IPs. It will only serve requests forwarded from you load balancer.


  # Source: [controller-role](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/templates/controller-role.yaml)
```
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
    name: k8s-ingress-nginx
    namespace: k8s-ingress-nginx
  rules:
    - apiGroups:
        - ''
      resources:
        - namespaces
      verbs:
        - get
    - apiGroups:
        - ''
      resources:
        - configmaps
        - pods
        - secrets
        - endpoints
      verbs:
        - get
        - list
        - watch
    - apiGroups:
        - ''
      resources:
        - services
      verbs:
        - get
        - list
        - update
        - watch
    - apiGroups:
        - extensions
        - networking.k8s.io   # k8s 1.14+
      resources:
        - ingresses
      verbs:
        - get
        - list
        - watch
    - apiGroups:
        - extensions
        - networking.k8s.io   # k8s 1.14+
      resources:
        - ingresses/status
      verbs:
        - update
    - apiGroups:
        - ''
      resources:
        - configmaps
      resourceNames:
        - ingress-controller-leader-nginx
      verbs:
        - get
        - update
    - apiGroups:
        - ''
      resources:
        - configmaps
      verbs:
        - create
    - apiGroups:
        - ''
      resources:
        - endpoints
      verbs:
        - create
        - get
        - update
    - apiGroups:
        - ''
      resources:
        - events
      verbs:
        - create
        - patch
  ---
```

  # Source: [controller-rolebinding](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/templates/controller-rolebinding.yaml)
```
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
    name: k8s-ingress-nginx
    namespace: k8s-ingress-nginx
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: k8s-ingress-nginx
  subjects:
    - kind: ServiceAccount
      name: nginx-ingress-serviceaccount
      namespace: k8s-ingress-nginx
  ---
```
  # Source: [controller-service](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/templates/controller-service.yaml)
```
  apiVersion: v1
  kind: Service
  metadata:
    annotations:
      # service.beta.kubernetes.io/aws-load-balancer-backend-protocol: http
      service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: '60'
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
      <!-- service.beta.kubernetes.io/aws-load-balancer-ssl-cert: arn:aws:acm:us-east-1:123456:certificate/xxxxxxxxxxxxx -->
      service.beta.kubernetes.io/aws-load-balancer-ssl-ports: https
      service.beta.kubernetes.io/aws-load-balancer-internal: "true"
      # service.beta.kubernetes.io/aws-load-balancer-type: elb
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
    name: ingress-nginx-controller
    namespace: k8s-ingress-nginx
  spec:
    type: LoadBalancer
    # externalTrafficPolicy: Local
    ports:
      - name: http
        port: 80
        protocol: TCP
        targetPort: http
      - name: https
        port: 443
        protocol: TCP
        targetPort: http
    selector:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  ---
```
  # Source: [controller-deployment](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/templates/controller-deployment.yaml)
```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
    name: ingress-nginx-controller
    namespace: k8s-ingress-nginx
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    revisionHistoryLimit: 10
    minReadySeconds: 0
    template:
      metadata:
        labels:
          app.kubernetes.io/name: ingress-nginx
          app.kubernetes.io/part-of: ingress-nginx
      spec:
        # dnsPolicy: ClusterFirst
        containers:
          - name: controller
            image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.33.0
            imagePullPolicy: IfNotPresent
            lifecycle:
              preStop:
                exec:
                  command:
                    - /wait-shutdown
            args:
              - /nginx-ingress-controller
              - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
              - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
              - --election-id=ingress-controller-leader
              - --ingress-class=nginx
              - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
              - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
              - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
              - --annotations-prefix=nginx.ingress.kubernetes.io
              # - --default-ssl-certificate=$(POD_NAMESPACE)/domain-cert
              # - --validating-webhook=:8443
              # - --validating-webhook-certificate=/usr/local/certificates/cert
              # - --validating-webhook-key=/usr/local/certificates/key
            securityContext:
              capabilities:
                drop:
                  - ALL
                add:
                  - NET_BIND_SERVICE
              runAsUser: 0
              # allowPrivilegeEscalation: true
            env:
              - name: POD_NAME
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.name
              - name: POD_NAMESPACE
                valueFrom:
                  fieldRef:
                    fieldPath: metadata.namespace
            livenessProbe:
              failureThreshold: 3
              httpGet:
                path: /healthz
                port: 10254
                scheme: HTTP
              initialDelaySeconds: 10
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 10
            readinessProbe:
              failureThreshold: 3
              httpGet:
                path: /healthz
                port: 10254
                scheme: HTTP
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 10
            ports:
              - name: http
                containerPort: 80
                protocol: TCP
              - name: https
                containerPort: 443
                protocol: TCP
              # - name: tohttps
              #   containerPort: 2443
              #   protocol: TCP
              # - name: webhook
              #   containerPort: 8443
              #   protocol: TCP
            # volumeMounts:
            #   - name: webhook-cert
            #     mountPath: /usr/local/certificates/
            #     readOnly: true
            resources:
              requests:
                cpu: 200m
                memory: 200Mi
        serviceAccountName: nginx-ingress-serviceaccount
        terminationGracePeriodSeconds: 300
        # volumes:
        #   - name: webhook-cert
        #     secret:
        #       secretName: ingress-nginx-admission
  ---
```
# Source: [controller-NetworkPolicy](https://github.com/snigdhasambitak/k8s-nginx-ingress-controller/blob/master/templates/controller-networkPolicies.yaml)
```
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: web-allow-all-ingress-service
  spec:
    podSelector:
      matchLabels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
    ingress:
    - {}

  ```

* Once deployed we can check the nginx ingress controller pods logs and make sure that the controller is running without any issues

```
kubectl logs chart-1590035165-nginx-ingress-controller-74565bb7b-2vjbj

-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       0.33.0
  Build:         git-446845114
  Repository:    https://github.com/kubernetes/ingress-nginx
  nginx version: nginx/1.17.10
```

* We can describe the ingress nginx service and verify if it is up and Running

```
kubectl  describe svc chart-1590035165-nginx-ingress-controller                                                          
```


# How to use the Nginx Ingress Controller

Assuming you’ve created the Ingress Controller above, your Ingress resources should be handled by the LoadBalancer created with the Ingress Controller service. Now no matter in which namepsace you create the ingress object, the backend is going to continuously probe for the annotation `kubernetes.io/ingress.class: "nginx"` and when it finds any ingress with the class as `nginx` then automatically it binds it to the controller and the services can be accessible using the ingress controller load balancers. This is independent of in which namespace you have your ingress object. As long as the ingress class is defined the controller can find the ingress and bind it to its backend.

As a quick test, you can deploy the following 2 deployments named `service-test` and `nginx`

## Service Requirements
As the ingress controller service already has a loadbalancer as a backend endpoint so the services that needs to use it should just be exposed as a ClusterIp with both the http and https ports opened. An example service and its related deployments can be found in the below sections. Once we create an ingress for this service then using the ingress controller load balancer we can access this service

## Deploy `service-test`

```
# Hello world Server Pod
kind: Deployment
apiVersion: apps/v1
metadata:
  name: service-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service_test_pod
  template:
    metadata:
      labels:
        app: service_test_pod
    spec:
      # serviceAccount: nginx-ingress-serviceaccount
      containers:
      - name: simple-http
        image: python:2.7
        imagePullPolicy: IfNotPresent
        command: ["/bin/bash"]
        args: ["-c", "echo \"<p>Hello from $(hostname)</p>\" > index.html; python -m SimpleHTTPServer 8080"]
        ports:
        - name: http
          containerPort: 8080

---
# Hello world Server Service
apiVersion: v1
kind: Service
metadata:
  name: service-test
spec:
  selector:
    app: service_test_pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080

---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: nginx-service
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - {}
```

## Deploy `nginx`

```
# Nginx deployments
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      # serviceAccountName: nginx-ingress-serviceaccount
      containers:
      - name: echoserver
        image: nginx
        ports:
        - containerPort: 80

---

# Nginx service
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - port: 80
    targetPort: 80
  # - port: 443
  #   targetPort: 80
  selector:
    app: nginx

---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: nginx-service
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - {}

---

```


##  Deploy `Test Ingress`

Now that we have the nginx deployment and service, we can deploy the ingress that binds the service to the ingress controller

```
--
# Hello world Server Ingress
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
   labels:
     app.kubernetes.io/name: nginx-ingress
     app.kubernetes.io/part-of: nginx-ingress
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: external-test.example.com
    http:
      paths:
      # - path: /test
      - backend:
          serviceName: service-test
          servicePort: 80
  - host: external-nginx.example.com
    http:
      paths:
      # - path: /nginx
      - backend:
          serviceName: nginx
          servicePort: 80
```

The important thing here to note is that we must use the following annotation for the ingress object to create an event in the controller backend:

```
kubernetes.io/ingress.class: "nginx"
```

This makes sure that when the ingress object is created the ingress controller listens to the ingress creation and binds it to its backend. At the end you can access the service nginx and service-test using the ingress loadbalancer

Once you have applied this ingress config you can describe the ingress to check if it points to the same loadbalancer that was created by the ingress nginx controller services

```
kubectl describe ingress test-ingress

Name: test-ingress
Namespace: k8s-layers-dev
Address: internal-a4e06a7ae1f544f6a8c1206199b82281-2108523565.us-east-1.elb.amazonaws.com
Default backend: default-http-backend:80 (<none>)
Rules:
Host Path Backends
---- ---- --------
 external-nginx.example.com            /nginx nginx:80 (100.1.2.6:80)
 external-test.example.com             /test service-test:80 (100.1.2.6:8080)

Annotations:
kubectl.kubernetes.io/last-applied-configuration: {"apiVersion":"networking.k8s.io/v1beta1","kind":"Ingress","metadata":{"annotations":{"kubernetes.io/ingress.class":"nginx","nginx.ingress.kubernetes.io/force-ssl-redirect":"true","nginx.ingress.kubernetes.io/rewrite-target":"/"},"name":"test-ingress","namespace":"k8s-layers-dev"},"spec":{"rules":[{"http":{"paths":[{"backend":{"serviceName":"service-test","servicePort":80},"path":"/test"},{"backend":{"serviceName":"nginx","servicePort":80},"path":"/nginx"}]}}]}}
kubernetes.io/ingress.class: nginx
nginx.ingress.kubernetes.io/force-ssl-redirect: true
nginx.ingress.kubernetes.io/rewrite-target: /
Events: <none>
```

##  Wiring it up and testing your Ingress

To test things out, you need to get your Ingress Controller entrypoint. For LoadBalancer services that will be:

```
kubectl get svc chart-1590035165-nginx-ingress-controller -o wide
```

If you match the Ingress rules, you will receive a default Nginx response:

```
LoadBalancer: curl -H 'Host:ingress-nginx.example.com' [ELB_DNS]
```

## Rewrite Targets
Rewriting can be controlled using the following annotations:

`nginx.ingress.kubernetes.io/rewrite-target` : `Target URI where the traffic must be redirected string`

`nginx.ingress.kubernetes.io/ssl-redirect` : `Indicates if the location section is accessible SSL only (defaults to True when Ingress contains a Certificate) bool`

`nginx.ingress.kubernetes.io/force-ssl-redirect` : `Forces the redirection to HTTPS even if the Ingress is not TLS Enabled	bool`

`nginx.ingress.kubernetes.io/app-root` : `Defines the Application Root that the Controller must redirect if it's in '/' context	string`

`nginx.ingress.kubernetes.io/use-regex` : `Indicates if the paths defined on an Ingress use regular expressions	bool`

### Examples

#### Rewrite Target

Create an Ingress rule with a rewrite annotation:

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
 annotations:
 nginx.ingress.kubernetes.io/rewrite-target: /$2
 name: rewrite
 namespace: default
spec:
 rules:
 - host: rewrite.bar.com
 http:
 paths:
 - backend:
 serviceName: http-svc
 servicePort: 80
 path: /something(/|$)(.*)

 ```

In this ingress definition, any characters captured by (.* ) will be assigned to the placeholder $2, which is then used as a parameter in the rewrite-target annotation.

For example, the ingress definition above will result in the following rewrites:

`rewrite.bar.com/something rewrites to rewrite.bar.com/`

`rewrite.bar.com/something/ rewrites to rewrite.bar.com/`

`rewrite.bar.com/something/new rewrites to rewrite.bar.com/new`


#### App Root

Create an Ingress rule with a app-root annotation:

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/app-root: /app1
  name: approot
  namespace: default
spec:
  rules:
  - host: approot.bar.com
    http:
      paths:
      - backend:
          serviceName: http-svc
          servicePort: 80
        path: /

```
Check the rewrite is working

```
$ curl -I -k http://approot.bar.com/
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.11.10
Date: Mon, 13 Mar 2017 14:57:15 GMT
Content-Type: text/html
Content-Length: 162
Location: http://stickyingress.example.com/app1
Connection: keep-alive
```        

# What problems does an nginx ingress controller solve?

As we have already seen in the above use cases, an nginx ingress controller solves multiple things and it is quite useful when you have a k8s cluster.

The Ingress resource supports the following features:

* Content-based routing:
   * `Host-based routing`: For example, routing requests with the host header foo.example.com to one group of services and the host header bar.example.com to another group.
   * `Path-based routing` : For example, routing requests with the URI that starts with /serviceA to service A and requests with the URI that starts with /serviceB to service B.
* TLS/SSL termination for each hostname, such as foo.example.com


# ingress + external-dns

external-dns.alpha.kubernetes.io/alias if set to true on an ingress, it will create an ALIAS record when the target is an ALIAS as well. To make the target an alias, the ingress needs to be configured correctly as described in the docs. In particular, the argument --publish-service=default/nginx-ingress-controller has to be set on the nginx-ingress-controller container. If one uses the nginx-ingress Helm chart, this flag can be set with the controller.publishService.enabled configuration option.

You can either directly specify the hostname using in the hosts section of the ingress like the below example. ExternalDns will create a A record of your subdomian, pointing to the ELB of the Ingress nginx service.

For example :
```
spec:
  rules:
  - host: external-nginx.example.com
    http:
      paths:
      # - path: /nginx
      - backend:
          serviceName: nginx
          servicePort: 80
```
Or if you want to use a global hostname for all of your ingress resources then you can use external-dns directly in the service annotations of your ingress controller

```
external-dns.alpha.kubernetes.io/hostname: ingress-nginx.example.com
```

# Nginx Controller + GRPC services

An AWS Network Load Balancer functions at the fourth layer of the Open Systems Interconnection (OSI) model. It can handle millions of requests per second. After the load balancer receives a connection request, it selects a target from the target group for the default rule. It attempts to open a TCP connection to the selected target on the port specified in the listener configuration.

In general, a gRPC client establishes a TCP connection that it holds open for as long as possible, and it makes all requests on that connection. When you use gRPC with a reverse-proxy-style load balancer (NLB), the load balancer will establish a single connection to one backend and use that connection for the life of the gRPC client.

The detailed example to achieve this is as follows:

https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/grpc

# References

https://kubernetes.github.io/ingress-nginx/deploy/#aws

https://kubernetes.github.io/ingress-nginx/deploy/rbac/

https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

https://aws.amazon.com/blogs/opensource/network-load-balancer-nginx-ingress-controller-eks/

https://kubernetes.github.io/ingress-nginx/examples/rewrite/

https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/grpc

https://kubernetes.github.io/ingress-nginx/user-guide/tls/

http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate

https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/enable-proxy-protocol.html

https://www.codegravity.com/blog/kubernetes-ingress-nginx-path-forwarding-does-not-work
