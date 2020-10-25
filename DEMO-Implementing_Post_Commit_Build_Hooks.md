<!-- Copy and paste the converted output. -->



# Implementing Post-Commit Build Hooks


## Prepare Repos


### 1. Preparing your Repo


```
$ git clone https://github.com/<ur own repo>/sunlife-training.git
```



### 2. Inspect the index.php file inside ~/sunlife-training/developer-src/trigger-builds/


```
$ cat ~/sunlife-training/developer-src/post-commit/index.php
```



```
<?php
print "Build hook application\n";
?>

```



## Create a new project


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



### 2. Create a new project &lt;urname>-post-commit


```
$ oc new-project sunny-post-commit
```



```
Now using project "sunny-post-commit" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Prepare build for manager Apps


### 1. Create new app 


```
$ oc new-app --name build-for-managers quay.io/redhattraining/builds-for-manager
```



### 2. Expose route


```
$ oc expose svc build-for-managers
```



### 3. Check the status


```
$ oc status
```



```
In project youruser-post-commit on server https://api.cluster.domain....

http://builds-for-managers... to pod port 8080-tcp (svc/builds-for-managers)
  dc/builds-for-managers deploys istag/builds-for-managers:latest
    deployment #1 deployed 5 minutes ago - 1 pod

...output omitted...
```



## Create new application


### 1. Create a new application from sources in Git


```
$ oc new-app --as-deployment-config  --name hook  php:7.3~http://github.com/<your username>/sunlife-trianing --context-dir developer-src/post-commit

```



### 2. Wait until the build finishes


```
$ oc logs -f bc/hook
```



### 3. Wait for the application to be ready and running


```
$  oc get pods
```



## Integrate the PHP application build with the builds-for-managers application


### 1. Run the command to create hook


```
$ oc set build-hook bc/hook --post-commit --command -- bash -c "curl -s -S -i -X POST http://<the route>/api/builds -f -d 'developer=\${DEVELOPER}&git=\${OPENSHIFT_BUILD_SOURCE}&project=\${OPENSHIFT_BUILD_NAMESPACE}'"

```



### 2. Verify that the post-commit build hook was created


```
$ oc describe bc/hook | grep Post
```



```
Post Commit Hook: ["bash", "-c", "\"curl -s -S -i -X POST http://builds-...
```



### 3. Start a new build using the oc start-build command with the -F option


```
$ oc start-build bc/hook -F
```



```
$ build.build.openshift.io/hook-2 started
...output omitted...
STEP 11: RUN bash -c "curl -s -S -i -X POST http://builds-for-managers-...
curl: (22) The requested URL returned error: 400 Bad Request
subprocess exited with status 22
subprocess exited with status 22
error: build error: error building at STEP "RUN bash -c "curl -s -S -i ...
```



### 4. List the builds and verify that one build failed due to a post-commit hook failure


```
$ oc get builds
```



```
NAME      TYPE      FROM          STATUS   ...output omitted...
hook-1    Source    Git@c2166cc   Complete
hook-2    Source    Git@c2166cc   Failed (GenericBuildFailed)
```



## Fix the missing environment variable problem.


### 1. Create the DEVELOPER build environment variable using your name with the oc set env command


```
$ oc set env bc/hook DEVELOPER="Your Name"
```



```
$ buildconfig.build.openshift.io/hook updated
```



### 2. Verify that the DEVELOPER environment variable is available in the hook build configuration


```
$ oc set env bc/hook --list
```



```
# buildconfigs hook
DEVELOPER=Your Name
```



### 3. Start a new build and display the logs to verify the HTTP API response code


```
$ oc start-build bc/hook -F
```



```
build.build.openshift.io/hook-3 started
...output omitted...
STEP 11: RUN bash -c "curl -s -S -i -X POST http://builds-for-managers-...
HTTP/1.1 200 OK
...output omitted...
Push successful
```



### 4. Get the builds-for-managers exposed route


```
$ oc get route/builds-for-managers -o jsonpath='{.spec.host}{"\n"}'
```



```
builds-for-managers-youruser-post-commit.apps.cluster.domain.example.com
```



### 5. Open a web browser to access the route


## Clean up


```
$ oc delete project  <urname>-post-commit
```

