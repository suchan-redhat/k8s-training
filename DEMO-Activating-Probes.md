<!-- Copy and paste the converted output. -->



# Activating Probes


## Create a new project, and then deploy the sample application


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



### 2. Create a new project &lt;urname>-probes


```
$ oc new-project sunny-s2i-probes
```



```
Now using project "sunny-probes" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Create a new application called probes from sources 


```
$ oc new-app --as-deployment-config  --name probes --build-env  nodejs:12~https://github.com/<ur name>/sunlife-training  --context-dir developer-src/probes

```



### 4. View the build logs. Wait until the build finishes.


```
$ oc logs -f bc/probes
```



### 5. Wait until the application is deployed


```
$ oc get pods
```



## Manually test the application's /ready and /healthz endpoints.


### 1. Use a route to expose the application to external access:


```
$ oc expose svc probes
```



### 2. Test the /ready


```
$ curl -i http://<the route>/ready
```



```
HTTP/1.1 503 Service Unavailable
...output omitted...
Error! Service not ready for requests...

HTTP/1.1 200 OK
...output omitted...
Ready for service requests...

```



### 3. Test the /healthz


```
$ curl -i http://<the route>/healthz
```



```
HTTP/1.1 200 OK
...output omitted...
OK
```



### 4. Test the app


```
$ curl  http://<the route>
```



```
Hello! This is the index page for the app.
```



## Activate readiness and liveness probes for the application


### 1. Use the oc set command to add liveness and readiness probes to the Deployment Config.


```
$ oc set probe dc/probes --liveness  --get-url=http://:8080/healthz  --initial-delay-seconds=2 --timeout-seconds=2

```



```
deploymentconfig.apps.openshift.io/probes probes updated
```



```
$ oc set probe dc/probes --readiness --get-url=http://:8080/ready  --initial-delay-seconds=2 --timeout-seconds=2
```



```
deploymentconfig.apps.openshift.io/probes probes updated
```



### 2. Verify the value in the livenessProbe and readinessProbe entries:


```
$ oc describe dc/probes
```



```
Name:		probes
...output omitted...
    Liveness:     http-get http://:8080/healthz delay=2s timeout=2s...
    Readiness:    http-get http://:8080/ready delay=2s timeout=2s...
...output omitted...
```



### 3. Wait for the application pod to redeploy and appear in a Running state


```
$ oc get pods
```



### 4. Use the oc logs command to see the results of the liveness and readiness probes:


```
$  oc logs -f dc/probes
```



```
...output omitted...
nodejs server running on http://0.0.0.0:8080
ping /healthz => pong [healthy]
ping /ready => pong [notready]
ping /healthz => pong [healthy]
ping /ready => pong [notready]
ping /healthz => pong [healthy]
ping /ready => pong [ready]
...output omitted...
```



## Simulate a failure of the liveness probe


### 1. In a different terminal window or tab run the command


```
$ xxxxs
```



### 2. Return to the terminal where you are monitoring the application deployment


```
$ oc logs -f dc/probes
```



```
...output omitted...
Received kill request. Changing app state to unhealthy...
ping /ready => pong [ready]
ping /healthz => pong [unhealthy]
ping /ready => pong [ready]
ping /healthz => pong [unhealthy]
ping /ready => pong [ready]
npm info lifecycle probes@1.0.0~poststart: probes@1.0.0
npm timing npm Completed in 150591ms
npm info ok
```



### 3. Verify that OpenShift restarts the unhealthy pod


```
$ oc get pod
```



### 4. Check the application logs again and note that the liveness probe succeeds and the application reports a healthy state


```
$ oc logs -f dc/probes
```



```
...output omitted...
ping /ready => pong [ready]
ping /healthz => pong [healthy]
...output omitted...
```



## Verify that the failure of the liveness probe is seen in the event log using the oc describe command on the pod from the previous step.


```
$ oc describe pod/probes-3-ltkkp

```



```
Events:
  Type     Reason     Age  From   Message
  ----     ------     ---- ----   -------
...output omitted...
  Warning  Unhealthy  ...  ...    Liveness probe failed: ... statuscode: 503
  Normal   Killing    ...  ...    Container ... will be restarted
  Normal   Created    ...  ...    Created container probes
  Warning  Unhealthy  ...  ...    Readiness probe failed: ... statuscode: 503

```



## Clean up


```
$ oc delete project  <urname>-apache-s2i
```


