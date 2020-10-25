<!-- Copy and paste the converted output. -->



# Injecting Configuration Data into an Application


## Review the application source code


### 1. Preparing your Repo


```
$ git clone https://github.com/<ur own repo>/sunlife-training.git
```



### 2. Inspect the app.js file


```
$ cat ~/sunlife-training/developer-src/app-config/app.js
```



```
// read in the APP_MSG env var
var msg = process.env.APP_MSG;
...output omitted...
// Read in the secret file
fs.readFile('/opt/app-root/secure/myapp.sec', 'utf8', function (secerr,secdata) { ...output omitted...
```



## Build and deploy a container to an OpenShift cluster


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



### 2. Create a new project &lt;urname>-app-config


```
$ oc new-project sunny-app-config
```



```
Now using project "sunny-app-config" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Build and deploy 


```
$ oc new-app --name myapp --build-env nodejs:10~https://github.com/<ur git name>/sunlife-training --context-dir developer-src/app-config

```



```
...output omitted...
--> Creating resources ...output omitted...
imagestream.image.openshift.io "myapp" created buildconfig.build.openshift.io "myapp" created deploymentconfig.apps.openshift.io "myapp" created service "myapp" created
--> Success
...output omitted...
```



### 4. View the build logs 


```
$ oc logs -f bc/myapp
```



```
Cloning "https:/github.com/youruser/DO288-apps#app-config" ...
...output omitted...
---> Installing application source ...
---> Building your Node application from source
...output omitted...
Pushing image image-registry.openshift-image-registry.svc:5000/youruser-app- config/myapp:latest ...
...output omitted...
Push successful
```



## Test the application


### 1. Wait until the application deploys 


```
$ oc get pods
```



### 2. Use a route to expose the application to external access:


```
$ oc expose svc myapp
```



```
route.route.openshift.io/myapp exposed
```



### 3. Identify the route URL


```
$ oc get route
```



```
NAME        HOST/PORT                                                                                             …
myapp        myapp-youruser-app-config.apps.cluster.domain.example.com              ...
```



### 4. Invoke the route URL identified


```
$ curl http://<route>

```



```
Value in the APP_MSG env var is => undefined
Error: ENOENT: no such file or directory, open '/opt/app-root/secure/myapp.sec'
```



## Create configmap and secret


### 1. Create a configuration map resource


```
$ oc create configmap myappconf   --from-literal APP_MSG="Test Message"
```



```
configmap/myappconf created
```



### 2. Verify that the configuration map contains the configuration data


```
$ oc describe cm/myappconf
```



```
Name: myappconf
...output omitted...
Data
====
APP_MSG:
----
Test Message ...output omitted...
```



### 3. Review the contents of the ~/sunlife-training/developer-src/app-config/ myapp.sec file


```
username=user1
password=pass1
salt=xyz123
```



### 3. Review the contents of the  ~/sunlife-training/developer-src/app-config/myapp.sec file


```
username=user1
password=pass1
salt=xyz123
```



### 3. Create a new secret


```
$ oc create secret generic myappfilesec  --from-file ~/sunlife-training/developer-src/app-config/myapp.sec
```



```
secret/myappfilesec created
```



### 4. Verify the contents of the secret


```
$ oc get secret/myappfilesec -o json
```



```
{
     "apiVersion": "v1", "data": { 
     "myapp.sec": "dXNlcm5hbWU9dXNlcjEKcGFzc3dvcmQ9cGFzczEKc2... },
     "kind": "Secret", "metadata": {
...output omitted...
     "name": "myappfilesec",
...output omitted...
     }, 
     "type": "Opaque"
}

```



## Inject the configuration map and the secret into the application container


### 1. Use the oc set env command to add the configuration map


```
$ oc set env dc/myapp --from configmap/myappconf
```



```
deploymentconfig.apps.openshift.io/myapp updated
```



### 2. Use the oc set volume command to add the secret to the deployment configuration


```
$ oc set volume dc/myapp --add  -t secret -m /opt/app-root/secure  --name myappsec-vol --secret-name myappfilesec
```



```
deploymentconfig.apps.openshift.io/myapp volume updated
```



## Verify that the application is redeployed and uses the data from the configuration map and the secret


### 1. Verify that the application is redeployed 


```
$ oc status
```



```
In project youruser-app-config on server ...output omitted...
http://myapp-youruser-app-config.apps.cluster.domain.example.com to pod port 8080- tcp (svc/myapp)
dc/myapp deploys istag/myapp:latest <-
bc/myapp source builds https://github.com/youruser/DO288-apps#app-config on
openshift/nodejs:10
deployment #3 deployed 24 seconds ago - 1 pod deployment #2 deployed 4 minutes ago deployment #1 deployed 18 minutes ago
```



### 2. Wait until the application pod is ready and in a Running state


```
$ oc get pods
```



### 3. Use the oc rsh command to inspect the environment variables


```
$ oc rsh myapp-3-wzdbh env | grep APP_MSG
```



```
APP_MSG=Test Message
```



### 4. Verify that the configuration map and secret were injected into the container


```
$ curl http://<the route>
```



```
Value in the APP_MSG env var is => Test Message
The secret is => username=user1
password=pass1
salt=xyz123
```



## Change the information stored in the configuration map and retest the application 


### 1. Use the oc edit configmap command to change the value of the APP_MSG key


```
$ oc edit cm/myappconf
```



```
...output omitted...
apiVersion: v1
data:
APP_MSG: Changed Test Message
kind: ConfigMap
...output omitted...
```



### 2. Verify that the value in the APP_MSG key is updated


```
$ oc describe cm/myappconf
```



```
Name: myappconf
...output omitted...
Data
====
APP_MSG:
----
Changed Test Message ...output omitted...
```



### 3. Use the oc rollout latest command to trigger a new deployment


```
$ oc rollout latest dc/myapp
```



```
deploymentconfig.apps.openshift.io/myapp rolled out.
```



### 4. Wait for the application pod to redeploy and appear in a Running state


```
$ oc get pods
```



### 5. Test the application and verify that the changed values in the configuration map display


```
$ curl http://<the route>
```



```
Value in the APP_MSG env var is => Changed Test Message
The secret is => username=user1
password=pass1
salt=xyz123
```



## Clean up


```
$ oc delete project  <urname>-app-config
```


