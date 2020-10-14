<!-- Copy and paste the converted output. -->



# Controlling Pod Scheduling Behavior


## Login to the OpenShift cluster and create the schedule-pods-&lt;urname> project.


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



### 2. Create the schedule-pods-&lt;urname> project.


```
$ oc new-project schedule-pods-sunny
```



```
Now using project "schedule-pods-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Deploy and scale a test application


### 1. Create a new application call hello


```
$ oc create deployment hello  --image quay.io/redhattraining/hello-world-nginx:v1.0
```



```
deployment.apps/hello created
```



### 2. Create a service for the application


```
$ oc expose deployment/hello --port 80 --target-port 8080
```



```
service/hello exposed
```



### 3. Make the service accessible by routes 


```
$ oc expose svc/hello
```



```
route.route.openshift.io/hello exposed
```



### 4. Manually scale the application 


```
$ oc scale --replicas 4 deployment/hello
```



```
deployment.apps/hello scaled
```



### 5. Verify the pods are distributed between worker nodes


```
$ oc get pods -o wide
```



```
NAME                   READY   STATUS    RESTARTS   AGE     IP             NODE                                              NOMINATED NODE   READINESS GATES
hello-d7fd6d8b-2lgn5   1/1     Running   0          76s     10.131.0.142   ip-10-0-223-34.ap-southeast-1.compute.internal    <none>           <none>
hello-d7fd6d8b-8r27t   1/1     Running   0          76s     10.130.4.66    ip-10-0-166-110.ap-southeast-1.compute.internal   <none>           <none>
hello-d7fd6d8b-9g55f   1/1     Running   0          3m26s   10.129.4.101   ip-10-0-211-223.ap-southeast-1.compute.internal   <none>           <none>
hello-d7fd6d8b-f9dr4   1/1     Running   0          76s     10.128.6.76    ip-10-0-141-135.ap-southeast-1.compute.internal   <none>           <none>
```



## Prepare worker node for load distribute


### 1. Login to Openshift As admin


```
$  oc login -u opentlc-mgr -p 'r3dh4t1!' https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443
```



```
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Login successful.

You have access to 273 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```



### 2. verify that none of the worker nodes use the env label.


```
$ oc get nodes -L env -l node-role.kubernetes.io/worker
```



```
NAME                                              STATUS     ROLES    AGE   VERSION           ENV
ip-10-0-141-135.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-142-187.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-142-60.ap-southeast-1.compute.internal    NotReady   worker   3d    v1.18.3+012b3ec   
ip-10-0-145-179.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-161-101.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-164-179.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-166-110.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-208-2.ap-southeast-1.compute.internal     Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-211-223.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-223-34.ap-southeast-1.compute.internal    Ready      worker   3d    v1.18.3+012b3ec   
```



### 3. Add the env=dev label to the first worker node to indicate that it is a development node.


```
$ oc label node  ip-10-0-141-135.ap-southeast-1.compute.internal  env=dev

```



```
node/ip-10-0-141-135.ap-southeast-1.compute.internal labeled  
```



### 4. Add the env=prod label to the second worker node to indicate that it is a development node.


```
$ oc label node ip-10-0-142-187.ap-southeast-1.compute.internal   env=prod

```



```
node/ip-10-0-142-187.ap-southeast-1.compute.internal labeled  
```



### 5. Verify the node has correct label


```
$  oc get nodes -L env -l node-role.kubernetes.io/worker

```



```
NAME                                              STATUS     ROLES    AGE   VERSION           ENV
ip-10-0-141-135.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   dev
ip-10-0-142-187.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   prod
ip-10-0-142-60.ap-southeast-1.compute.internal    NotReady   worker   3d    v1.18.3+012b3ec   
ip-10-0-145-179.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-161-101.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-164-179.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-166-110.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-208-2.ap-southeast-1.compute.internal     Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-211-223.ap-southeast-1.compute.internal   Ready      worker   3d    v1.18.3+012b3ec   
ip-10-0-223-34.ap-southeast-1.compute.internal    Ready      worker   3d    v1.18.3+012b3ec   
```



## Switch back to developer and modify hello application so that it is deployed to the development node.


### 1. Login to Openshift As developer


```
$  oc login -u user1 -p 'r3dh4t1!' https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443
```



```
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Login successful.

You have access to 273 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```



### 2. Modify the deployment 


```
$ oc edit deployment/hello
```



```
...output omitted...
terminationMessagePath: /dev/termination-log
terminationMessagePolicy: File dnsPolicy: ClusterFirst nodeSelector:
env: dev
      restartPolicy: Always
...output omitted...   
```



### 3. Verify the pods are located on the same node


```
$ oc get pods -o wide
```



```
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE                                              NOMINATED NODE   READINESS GATES
hello-995785cf5-7cvl8   1/1     Running   0          83s   10.128.6.78   ip-10-0-141-135.ap-southeast-1.compute.internal   <none>           <none>
hello-995785cf5-95s8s   1/1     Running   0          80s   10.128.6.79   ip-10-0-141-135.ap-southeast-1.compute.internal   <none>           <none>
hello-995785cf5-gf8wf   1/1     Running   0          80s   10.128.6.80   ip-10-0-141-135.ap-southeast-1.compute.internal   <none>           <none>
hello-995785cf5-qkpl8   1/1     Running   0          83s   10.128.6.77   ip-10-0-141-135.ap-southeast-1.compute.internal   <none>           <none>  
```



### 4. Remove the schedule-pod-sunny project


```
$ oc delete project schedule-pods-sunny 
```



```
project.project.openshift.io "schedule-pods-sunny" deleted
```



## Clean up by removing the env label


### 1. Login to Openshift As admin


```
$  oc login -u opentlc-mgr -p 'r3dh4t1!' https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443
```



```
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Login successful.

You have access to 273 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```



### 2. Remove the env label from all worker nodes


```
$ oc label node -l node-role.kubernetes.io/worker env-
```



```
node/ip-10-0-141-135.ap-southeast-1.compute.internal labeled
node/ip-10-0-142-187.ap-southeast-1.compute.internal labeled
label "env" not found.
node/ip-10-0-142-60.ap-southeast-1.compute.internal not labeled
label "env" not found.
node/ip-10-0-145-179.ap-southeast-1.compute.internal not labeled
label "env" not found.
node/ip-10-0-161-101.ap-southeast-1.compute.internal not labeled
label "env" not found.
node/ip-10-0-164-179.ap-southeast-1.compute.internal not labeled
label "env" not found.
node/ip-10-0-166-110.ap-southeast-1.compute.internal not labeled
label "env" not found.
node/ip-10-0-208-2.ap-southeast-1.compute.internal not labeled
label "env" not found.
node/ip-10-0-211-223.ap-southeast-1.compute.internal not labeled
label "env" not found.
node/ip-10-0-223-34.ap-southeast-1.compute.internal not labeled  
```


