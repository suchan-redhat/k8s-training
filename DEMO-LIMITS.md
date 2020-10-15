<!-- Copy and paste the converted output. -->



# Limiting Resource Usage


## Login to the OpenShift cluster and create the schedule-limit-&lt;urname> project.


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



### 2. Create the schedule-limit-&lt;urname> project.


```
$ oc new-project schedule-limit-sunny
```



```
Now using project "schedule-pods-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Deploy and scale a test application


### 1. Create a deployment resources file and save it to temp


```
$ oc create deployment hello-limit  --image quay.io/redhattraining/hello-world-nginx:v1.0  --dry-run -o yaml > /tmp/hello-limit.yaml
```



### 2. Edit the hello-limit and replace the resource part


```
$ vi /tmp/hello-limit.yaml
```



```

...output omitted...
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-limit
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-limit
    spec:
      containers:
      - image: quay.io/redhattraining/hello-world-nginx:v1.0
        name: hello-world-nginx
        resources: 
          requests:
            cpu: "17"
            memory: 20Mi
status: {}
```



### 3. Create the new application using the resource file


```
$ oc create --save-config -f /tmp/hello-limit.yaml 
```



```
deployment.apps/hello-limit created
```



### 4. Pods are in pending status


```
$ oc get pods
```



```
NAME                           READY   STATUS    RESTARTS   AGE
hello-limit-6565dc5564-9r7nl   0/1     Pending   0          26s
```



### 5. The pod cannot schedule due to not sufficient CPU


```
$ oc get events --field-selector type=Warning
```



```
LAST SEEN   TYPE      REASON             OBJECT                             MESSAGE
<unknown>   Warning   FailedScheduling   pod/hello-limit-6565dc5564-9r7nl   0/13 nodes are available: 13 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-6565dc5564-9r7nl   0/13 nodes are available: 13 Insufficient cpu.
```



## Redeploy the application so that its requesting fewer CPU


### 1. Edit the hello-limit and replace the resource part


```
$  vi /tmp/hello-limit.yaml
```



```
… out put omitted … 
    spec:
      containers:
      - image: quay.io/redhattraining/hello-world-nginx:v1.0
        name: hello-world-nginx
        resources:
          requests:
            cpu: "10"
            memory: 20Mi
status: {}

```



### 2. Apply the changes


```
$ oc apply -f /tmp/hello-limit.yaml
```



```
deployment.apps/hello-limit configured  
```



### 3. Verify the pod is running 


```
$ oc get pods

```



```
NAME                          READY   STATUS    RESTARTS   AGE
hello-limit-bd75fb9b6-5jjlr   1/1     Running   0          25s
```



## Attempt to scale your application to 8 pods. After verifying this will exceed your cluster limit


### 1. Manually scale hello-limit application to 16 pods


```
$  oc scale --replicas 16 deployment/hello-limit
```



```
deployment.apps/hello-limit scaled
```



### 2. Check if all pods are running 


```
$ oc get pods
```



```
NAME                          READY   STATUS              RESTARTS   AGE
hello-limit-bd75fb9b6-4dx57   0/1     ContainerCreating   0          25s
hello-limit-bd75fb9b6-5jjlr   1/1     Running             0          3m21s
hello-limit-bd75fb9b6-6mq62   1/1     Running             0          64s
hello-limit-bd75fb9b6-8tt4d   0/1     Pending             0          25s
hello-limit-bd75fb9b6-bhcxr   1/1     Running             0          64s
hello-limit-bd75fb9b6-cjtjg   0/1     Pending             0          25s
hello-limit-bd75fb9b6-d68vx   1/1     Running             0          64s
hello-limit-bd75fb9b6-d8zpf   0/1     Pending             0          25s
hello-limit-bd75fb9b6-gmrbh   1/1     Running             0          64s
hello-limit-bd75fb9b6-jjg6d   0/1     Pending             0          25s
hello-limit-bd75fb9b6-klbq2   0/1     Pending             0          25s
hello-limit-bd75fb9b6-l6spw   0/1     Pending             0          25s
hello-limit-bd75fb9b6-qn29c   1/1     Running             0          64s
hello-limit-bd75fb9b6-qpnp7   1/1     Running             0          64s
hello-limit-bd75fb9b6-wkmsq   0/1     Pending             0          25s
hello-limit-bd75fb9b6-xf47t   1/1     Running             0          64s
```



### 3. Verify that one or more pods cannot schedule due to in sufficient CPU


```
$ oc get events --field-selector type=Warning
```



```
LAST SEEN   TYPE      REASON             OBJECT                             MESSAGE
<unknown>   Warning   FailedScheduling   pod/hello-limit-6565dc5564-9r7nl   0/13 nodes are available: 13 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-6565dc5564-9r7nl   0/13 nodes are available: 13 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-6565dc5564-9r7nl   skip schedule deleting pod: schedule-limit-sunny/hello-limit-6565dc5564-9r7nl
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-8tt4d    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-8tt4d    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-cjtjg    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-cjtjg    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-d8zpf    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-d8zpf    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-jjg6d    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-jjg6d    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-klbq2    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-klbq2    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-l6spw    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-l6spw    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-wkmsq    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
<unknown>   Warning   FailedScheduling   pod/hello-limit-bd75fb9b6-wkmsq    0/13 nodes are available: 1 node(s) had taint {node.kubernetes.io/unreachable: }, that the pod didn't tolerate, 12 Insufficient cpu.
```



### 4. Delete all the resources 


```
$ oc delete all -l app=hello-limit
```



```
pod "hello-limit-bd75fb9b6-4dx57" deleted
pod "hello-limit-bd75fb9b6-5jjlr" deleted
pod "hello-limit-bd75fb9b6-6mq62" deleted
pod "hello-limit-bd75fb9b6-8tt4d" deleted
pod "hello-limit-bd75fb9b6-bhcxr" deleted
pod "hello-limit-bd75fb9b6-cjtjg" deleted
pod "hello-limit-bd75fb9b6-d68vx" deleted
pod "hello-limit-bd75fb9b6-d8zpf" deleted
pod "hello-limit-bd75fb9b6-gmrbh" deleted
pod "hello-limit-bd75fb9b6-jjg6d" deleted
pod "hello-limit-bd75fb9b6-klbq2" deleted
pod "hello-limit-bd75fb9b6-l6spw" deleted
pod "hello-limit-bd75fb9b6-qn29c" deleted
pod "hello-limit-bd75fb9b6-qpnp7" deleted
pod "hello-limit-bd75fb9b6-wkmsq" deleted
pod "hello-limit-bd75fb9b6-xf47t" deleted
deployment.apps "hello-limit" deleted
```



## Deploy a Second APP to test memory usage


### 1. Prepare the below yaml and save it to /tmp/loadmem.yaml


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
          resources:
            requests:
              cpu: "100m"
              memory: 20M
            limits:
              cpu: "500m"
              memory: 200M
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



### 2. Create the application using the prepared resource 


```
$  oc create --save-config  -f /tmp/loadmem.yaml
```



```
deployment.apps/loadtest created
service/loadtest created
route.route.openshift.io/loadtest created
```



### 3. Obtain the route of the load app


```
$ oc get route
```



```
NAME       HOST/PORT                                                                                      PATH   SERVICES   PORT   TERMINATION   WILDCARD
loadtest   loadtest-schedule-limit-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com          loadtest   8080                 None
```



### 4. Open two terminal run the below commands respectively


```
$ watch oc get pods
```



```
$ watch oc adm top pod
```



## Generate some loading by application API


### 1. Use the application API to increase the memory loadby 150MB for 60 seconds


```
$ curl -X GET  http://loadtest-schedule-limit-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/api/loadtest/v1/mem/150/60
```



### 2. Observe the output of the two screen


## Generate loading by application API which container cannot handle


### 1. Use the application API to increase the memory loadby 150MB for 60 seconds


```
$ curl -X GET  http://loadtest-schedule-limit-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/api/loadtest/v1/mem/200/60
```



### 2. Observe the output of the two screen


```
NAME                        READY   STATUS      RESTARTS   AGE
loadtest-58d967ffbb-wpslj   0/1     OOMKilled   3          10m
```



## Create quota for the project


### 1. Login to OCP as administrator


```
$ oc login -u opentlc-mgr -p 'r3dh4t1!' https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443
```



```
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Login successful.

You have access to 273 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```



### 2. Create a quota for project schedule-limit-sunny


```
$ oc create quota project-quota  --hard cpu="3",memory="1G",configmaps="3"  -n schedule-limit-sunny

```



```
$ resourcequota/project-quota created
```



## As developer try to exceed the configmap quota


### 1. Login to OCP as Developer


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



### 2. Use a loop to create 4 configmap


```
$  for X in {1..4}; do oc create configmap my-config${X} --from-literal key${X}=value${X}; done

```



```
configmap/my-config1 created
configmap/my-config2 created
configmap/my-config3 created
Error from server (Forbidden): configmaps "my-config4" is forbidden: exceeded quota: project-quota, requested: configmaps=1, used: configmaps=3, limited: configmaps=3

You have access to 273 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```



## Clean up


```
$ oc login -u opentlc-mgr -p 'r3dh4t1!' https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443

$ oc delete resourcequota project-quota

$ oc delete project schedule-limit-sunny
```


