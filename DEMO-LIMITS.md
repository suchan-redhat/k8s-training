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


