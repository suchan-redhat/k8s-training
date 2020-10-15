<!-- Copy and paste the converted output. -->



# Limiting Resource Usage


## Login to the OpenShift cluster and create the schedule-scale-&lt;urname> project.


### 1. Login to OCP as developer


```
$ oc login -u user1 -p 'r3dh4t1!' https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443
```



```
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Login successful.

You have access to 273 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```



### 2. Create the schedule-scale-&lt;urname> project.


```
$ oc new-project schedule-scale-sunny
```



```
Now using project "schedule-scale-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Deploy a test application


### 1. Prepare a Load Test app

Create below yaml file and save it to /tmp/loadtest.yaml


```
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    creationTimestamp: null
    labels:
      app: loadtest
    name: loadtest
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: loadtest
    strategy: {}
    template:
      metadata:
        creationTimestamp: null
        labels:
          app: loadtest
      spec:
        containers:
        - image: quay.io/redhattraining/loadtest:v1.0
          name: loadtest
          resources: {}
  status: {}
- apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: null
    labels:
      app: loadtest
    name: loadtest
  spec:
    ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
    selector:
      app: loadtest
  status:
    loadBalancer: {}
- apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    annotations:
      haproxy.router.openshift.io/timeout: 60s
    creationTimestamp: null
    labels:
      app: loadtest
    name: loadtest
  spec:
    host: ""
    port:
      targetPort: 8080
    subdomain: ""
    to:
      kind: ""
      name: loadtest
      weight: null
  status:
    ingress: null
kind: List
metadata: {}
```



### 2. Modify the file for resource limiting


```
$ vi /tmp/loadtest.yaml
```



```
...output omitted...
spec:
containers:
- image: quay.io/redhattraining/loadtest:v1.0
        name: loadtest
        resources:
            requests: 
               cpu: "25m"
               memory: 25Mi
          limits:
status: {}
```



### 3. Create the new application using the resource file


```
$ oc create --save-config -f /tmp/loadtest.yaml 
```



```
deployment.apps/loadtest created
service/loadtest created 
route.route.openshift.io/loadtest created
```



### 4. Pods are in Running status


```
$ oc get pods
```



```
NAME                        READY   STATUS    RESTARTS   AGE
loadtest-5797b988dc-j2wk4   1/1     Running   0          59s
```



##  Manually Scale your application


### 1. Scale the loadtest deployment up to five pods


```
$ oc scale --replicas 5 deployment/loadtest
```



```
deployment.extensions/loadtest scaled
```



### 2. Verify that all five pods are running 


```
$ oc get pods
```



```
NAME                        READY   STATUS    RESTARTS   AGE
loadtest-5797b988dc-f6t6k   1/1     Running   0          59s
loadtest-5797b988dc-j2wk4   1/1     Running   0          2m3s
loadtest-5797b988dc-jlrkq   1/1     Running   0          59s
loadtest-5797b988dc-m4dpv   1/1     Running   0          59s
loadtest-5797b988dc-mc9jp   1/1     Running   0          59s
```



### 3. Scale loadtest deployment back to one pod


```
$ oc scale --replicas 1 deployment/loadtest

```



```
deployment.extensions/loadtest scaled
```



### 3. Verify only one pod is running 


```
$ oc get pods

```



```
NAME                        READY   STATUS    RESTARTS   AGE
loadtest-5797b988dc-j2wk4   1/1     Running   0          3m50s
```



## Config the loadtest application to automatically scale


### 1. Create a horizontal pod autoscaler


```
$  oc autoscale deployment/loadtest --min 2 --max 10 --cpu-percent 50
```



```
horizontalpodautoscaler.autoscaling/loadtest autoscaled
```



### 2. Get the route to allow controlling CPU by REST


```
$ oc get route/loadtest
```



```
NAME       HOST/PORT                                                                                      PATH   SERVICES   PORT   TERMINATION   WILDCARD
loadtest   loadtest-schedule-scale-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com          loadtest   8080                 None
```



### 3. Simulate CPU load by rest


```
$ curl -X GET  http://loadtest-schedule-scale-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/api/loadtest/v1/cpu/16

```



```

```



### 4. Open a second Terminal and monitor the status


```
$ watch oc get hpa/loadtest
```



```

```



## Create a second application call scaling


### 1. Create the application using oc new-app


```
$  oc new-app  --docker-image quay.io/redhattraining/scaling:v1.0
```



```
--> Found container image 4e17b8d (11 months old) from quay.io for "quay.io/redhattraining/scaling:v1.0"

    Apache 2.4 with PHP 7.2 
    ----------------------- 
    PHP 7.2 available as container is a base platform for building and running various PHP 7.2 applications and frameworks. PHP is an HTML-embedded scripting language. PHP attempts to make it easy for developers to write dynamically generated web pages. PHP also offers built-in database integration for several commercial and non-commercial database management systems, so writing a database-enabled webpage with PHP is fairly simple. The most common use of PHP coding is probably as a replacement for CGI scripts.

    Tags: builder, php, php72, rh-php72

    * An image stream tag will be created as "scaling:v1.0" that will track this image
    * This image will be deployed in deployment config "scaling"
    * Ports 8080/tcp, 8443/tcp will be load balanced by service "scaling"
      * Other containers can access this service through the hostname "scaling"

--> Creating resources ...
    imagestream.image.openshift.io "scaling" created
    deploymentconfig.apps.openshift.io "scaling" created
    service "scaling" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/scaling' 
    Run 'oc status' to view your app.

```



### 2. Create a route for this app


```
$ oc expose svc/scaling
```



```
route.route.openshift.io/scaling exposed
```



### 3. Scale the deploymentconfig to 3 pods


```
$ oc scale --replicas 3 dc/scaling
```



```
deploymentconfig.apps.openshift.io/scaling scaled
```



### 4. Verify all Pods are running


```
$ oc get pods -o wide -l deploymentconfig=scaling
```



```
NAME              READY   STATUS    RESTARTS   AGE     IP             NODE                                              NOMINATED NODE   READINESS GATES
scaling-1-br6wc   1/1     Running   0          66s     10.128.6.84    ip-10-0-141-135.ap-southeast-1.compute.internal   <none>           <none>
scaling-1-mbqn7   1/1     Running   0          66s     10.130.4.71    ip-10-0-166-110.ap-southeast-1.compute.internal   <none>           <none>
scaling-1-rjx2l   1/1     Running   0          2m49s   10.131.0.146   ip-10-0-223-34.ap-southeast-1.compute.internal    <none>           <none>
```



### 5. Display the route


```
$ oc get route/scaling
```



```
NAME      HOST/PORT                                                                                     PATH   SERVICES   PORT       TERMINATION   WILDCARD
scaling   scaling-schedule-scale-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com          scaling    8080-tcp                 None
```



### 6. Send some request and observe response


```
$ (for x in {1..100}; do curl -s  http://scaling-schedule-scale-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com; done) | sort | uniq -c

```



```
     34 Server IP: 10.128.6.84 
     33 Server IP: 10.130.4.71 
     33 Server IP: 10.131.0.146 
```



## Revisit the autoscaling app

Check and see if the pod back down to 2 Pods


```
$  oc get hpa/loadtest
```



```
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
loadtest   Deployment/loadtest   0%/50%    2         10        2          21m

```



## Clean up by delete projects


```
$  oc delete project schedule-scale-sunny
```



```
project.project.openshift.io "schedule-scale-sunny" deleted
```


