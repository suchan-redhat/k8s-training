<!-- Copy and paste the converted output. -->



# Implementing a Deployment Strategy


## Review the application source code


### 1. Preparing your Repo


```
$ git clone https://github.com/<ur own repo>/sunlife-training.git
```



### 2. Inspect the Quip.java file inside ~/sunlife-training/developer-src/quip/src/main/java/com/redhat/training/example


```
$ cat ~/sunlife-training/developer-src/quip/src/main/java/com/redhat/training/example/Quip.java
```



```
...output omitted...
@Path("/")
public class Quip {

@GET
@Produces("text/plain")
public Response index() throws Exception {
    String host = InetAddress.getLocalHost().getHostName();
    return Response.ok("Veni, vidi, vici...\n").build();1
  }

@GET
@Path("/ready")
@Produces("text/plain")
public Response ready() throws Exception {
    return Response.ok("OK\n").build();2
  }
...output omitted...
```



## Create an application based on the source code 


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



### 2. Create a new project &lt;urname>-app-deploy


```
$ oc new-project sunny-app-deploy
```



```
Now using project "sunny-app-deploy" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Create an application using the oc new app command


```
$ oc new-app --as-deployment-config --name quip   -i redhat-openjdk18-openshift:1.5  https://github.com/suchan-redhat/sunlife-training  --context-dir developer-src/quip


```



```
--> Found image c1bf724 (9 months old) in image stream "openshift/redhat-openjdk18-...
--> Creating resources ...
    imagestream.image.openshift.io "quip" created
    buildconfig.build.openshift.io "quip" created
    deploymentconfig.apps.openshift.io "quip" created
    service "quip" created
--> Success
...output omitted...
```



### 4. View the application build logs


```
$ oc logs -f bc/quip
```



```
...output omitted...
[INFO] Repackaged .war: /tmp/src/target/quip.war
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
...output omitted...
Pushing image ...-registry.svc:5000/youruser-app-deploy/quip:latest ...
...output omitted...
Push successful
```



### 5. Wait until the application is deployed


```
$ oc get pod
```



```
...output omitted...
[INFO] Repackaged .war: /tmp/src/target/quip.war
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
...output omitted...
Pushing image ...-registry.svc:5000/youruser-app-deploy/quip:latest ...
...output omitted...
Push successful
```



## Test the application to verify that it serves requests from clients


### 1. Review the application logs to see if there were any errors during startup


```
$ oc logs dc/quip
```



```
...output omitted...
... INFO  [org.jboss.as.server] (main) WFLYSRV0010: Deployed "quip.war" (...)
... INFO  [org.wildfly.swarm] (main) WFSWARM99999: Thorntail is Ready

```



### 2. Verify that the OpenShift service for the application has a registered endpoint to route incoming requests


```
$ oc describe svc/quip
```



```
Name:              quip
Namespace:         youruser-app-deploy
Labels:            app=quip
Annotations:       openshift.io/generated-by: OpenShiftNewApp
Selector:          app=quip,deploymentconfig=quip
Type:              ClusterIP
IP:                172.30.37.127
Port:              8080-tcp  8080/TCP
TargetPort:        8080/TCP
Endpoints:         10.128.2.111:8080
...output omitted...
```



### 3. Expose the application for external access by using a route


```
$ oc expose svc quip
```



```
route.route.openshift.io/quip exposed
```



### 4. Obtain the route URL using the oc get route command


```
$ oc get route/quip  -o jsonpath='{.spec.host}{"\n"}'
```



```
quip-youruser-app-deploy.apps.cluster.domain.example.com
```



### 5. Test the application using the route URL you obtained in the previous step


```
$ curl  http://<the route>
```



```
Veni, vidi, vici...
```



## Activate readiness and liveness probes for the application


### 1. Use the oc set command to add a liveness and readiness probe to the Deployment Config


```
$ oc set probe dc/quip --liveness  --get-url=http://:8080/ready  --initial-delay-seconds=30 --timeout-seconds=2
```



```
deploymentconfig.apps.openshift.io/quip probes updated
```



```
$ oc set probe dc/quip --readiness  --get-url=http://:8080/ready  --initial-delay-seconds=30 --timeout-seconds=2
```



```
deploymentconfig.apps.openshift.io/quip probes updated
```



### 2. Verify the value in the livenessProbe and readinessProbe entries


```
$ oc describe dc/quip
```



```
Name:		quip
...output omitted...
    Liveness:    http-get http://:8080/ready delay=30s timeout=2s...
    Readiness:   http-get http://:8080/ready delay=30s timeout=2s...
...output omitted...
```



### 3. Wait for the application pod to redeploy and appear in a Running state


```
$ oc get pods
```



### 4. Use the oc describe command to inspect the running pod and ensure the probes are active


```
$ oc describe pod quip-3-hppqxi | grep http-get
```



```
    Liveness:    http-get http://:8080/ready delay=30s timeout=2s...
    Readiness:   http-get http://:8080/ready delay=30s timeout=2s...
```



### 5. Test the application using the route URL you obtained in the previous step


```
$ curl  http://<the route>

```



```
Veni, vidi, vici...
```



## Make some changes to the application source. Verify that you can see the changes when you test the application.


### 1. Run the below script


```
sed -i 's/Veni, vidi, vici/I came, I saw, I conquered/g' \
~/sunlife-training/developer-src/quip/src/main/java/com/redhat/training/example/Quip.java

echo "Committing the changes..."
cd ~/sunlife-training/developer-src/quip/
git commit -a -m "Changed quip lang to english"

echo "Pushing changes to classroom Git repository..."
git push
cd ~
```



### 2. Start a new build of the application and follow the build logs


```
$ oc start-build quip -F
```



```
build.build.openshift.io/quip-2 started
...output omitted...
Push successful
```



### 3. Wait until a new application pod is deployed


```
$ oc get pods
```



### 4. Retest the application after the change, and verify that a message in English is printed:


```
$ curl http://<the route>
```



```
I came, I saw, I conquered...
```



## Roll back to the previous deployment. Verify that you see the older message 


### 1. Roll back to the previous version of the deployment


```
$ oc rollback dc/quip
```



```
deploymentconfig.apps.openshift.io/quip deployment #5 rolled back to quip-3
Warning: the following images triggers were disabled: quip:latest
You can re-enable them with: oc set triggers dc/quip --auto

```



### 2. Wait for the new application pod to deploy


```
$ oc get pods
```



```
NAME           READY     STATUS      RESTARTS   AGE
quip-1-build    0/1     Completed   0          6m
quip-1-deploy   0/1     Completed   0          6m
quip-3-build    0/1     Completed   0          4m
quip-3-deploy   0/1     Completed   0          4m
quip-4-deploy   0/1     Completed   0          2m
quip-5-deploy   0/1     Completed   0          45s
quip-5-z7lg5    1/1     Running     0          17s
```



### 3. After you have rolled back the application, retest it and verify that the Latin message is printed


```
$ curl http://<the route>
```



```
Veni, vidi, vici...
```



## Clean up


```
$ oc delete project  <urname>-app-deploy
```


