<!-- Copy and paste the converted output. -->



# Triggering Builds


## Prepare Repos


### 1. Preparing your Repo


```
$ git clone https://github.com/<ur own repo>/sunlife-training.git
```



### 2. Inspect the index.php file inside ~/sunlife-training/developer-src/trigger-builds/


```
$ cat ~/sunlife-training/developer-src/trigger-builds/index.php
```



```
<?php
print "Trigger builds with image stream \n";
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



### 2. Create a new project &lt;urname>-trigger-builds


```
$ oc new-project sunny-trigger-builds
```



```
Now using project "sunny-trigger-builds" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Prepare Images


### 1. Copy an image from docker to quay.io


```
$  podman login -u <your login id> quay.io
$ skopeo copy  docker://docker.io/centos/php-72-centos7:latest docker://quay.io/${RHT_OCP4_QUAY_USER}/php-70-rhel7:latest

```



```
Password:
Login Succeeded!
Getting image source signatures
...output omitted...
Writing manifest to image destination
Storing signatures
```



### 2. Create pull secret


```
$ oc create secret generic quay-registry  --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json --type kubernetes.io/dockerconfigjson
$ oc secrets link builder quay-registry
```



```
$ secret/quay-registry created
```



### 4. Update the php image stream to fetch the metadata for the new container image


```
$ oc import-image php  --from quay.io/${RHT_OCP4_QUAY_USER}/php-70-rhel7 --confirm
```



```
imagestream.image.openshift.io/php-70-rhel7 imported
```



## Create new application


### 1. Create a new application from sources in Git


```
$ oc new-app --name trigger php~http://github.com/<your username>/sunlife-trianing --context-dir developer-src/trigger-builds

```



```
--> Found image ... "youruser-trigger-builds/php" ... for "php"
...output omitted...
--> Success
Build scheduled, use 'oc logs -f bc/trigger' to track its progress.
...output omitted...
```



### 2. Wait until the build finishes


```
$ oc logs -f bc/trigger
```



### 3. Wait for the application to be ready and running


```
$  oc get pods
```



### 4. Verify that an image change trigger is defined as part of the build configuration


```
$  oc describe bc/trigger | grep Triggered
```



```
Triggered by: Config, ImageChange
```



## Update the image stream to start a new build


### 1. Upload the new version of the PHP S2I builder image to the Quay.io registry


```
$ mkdir /tmp/newphp
$ cat <<EOF >>  /tmp/newphp/Dockerfile
FROM docker.io/centos/php-72-centos7:latest 
ENV VERSION=2.0
EOF
$ cd /tmp/newphp/
$ buildah build-using-dockerfile -t quay.io/${RHT_OCP4_QUAY_USER}/php-70-rhel7:latest
$ podman push quay.io/${RHT_OCP4_QUAY_USER}/php-70-rhel7:latest
```



### 2. Update the php image stream to fetch the metadata 


```
$ oc import-image php
```



```
imagestream.image.openshift.io/php-70-rhel7 imported
```



### 3. Check the new build


```
$ oc get builds
```



```
$ oc describe build trigger-2 | grep cause
```



## Clean up


```
$ oc delete project  <urname>-trigger-builds
```


* delete images from quay.io as well

