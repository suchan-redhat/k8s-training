<!-- Copy and paste the converted output. -->



# Advanced Dockerfile Instructions


## Review the application source code


### 1. Review the below Dockfile as the prebuilt image in Quay.io


```
FROM registry.access.redhat.com/ubi8/ubi:8.0

MAINTAINER Red Hat Training <training@redhat.com>

# DocumentRoot for Apache
ENV DOCROOT=/var/www/html

RUN yum install -y --no-docs --disableplugin=subscription-manager httpd && \
         yum clean all --disableplugin=subscription-manager -y && \
         echo "Hello from the httpd-parent container!" > ${DOCROOT}/index.html

# Allows child images to inject their own content into DocumentRoot
ONBUILD COPY src/ ${DOCROOT}/

EXPOSE 80

# This stuff is needed to ensure a clean start RUN rm -rf /run/httpd && mkdir /run/httpd
# Run as the root user
USER root

# Launch httpd
CMD /usr/sbin/httpd -DFOREGROUND

```



### 2. Preparing your Repo


```
$ git clone https://github.com/<ur own repo>/sunlife-training.git
```



### 3. Inspect the docker file at


```
$ cat ~/sunlife-training/developer-src/container-build/Dockerfile
```



```
$ FROM quay.io/redhattraining/http-parent
```



### 4. The child container provides its own index.html


```
$ cat ~/sunlife-training/developer-src/container-build/src/index.html
```



```
<!DOCTYPE html>
<html>
<body>
  Hello from the Apache child container!
</body>
</html>
```



## Build and deploy a container to an OpenShift clustera


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



### 2. Create a new project &lt;urname>-container-build


```
$ oc new-project sunny-container-build
```



```
Now using project "sunny-container-build" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Build and deploy the Apache HTTP Server child image


```
$ oc new-app --name hola  https://github.com/<urname>/sunlife-training --context-dir developer-scr/container-build

```



```
--> Found Docker image 1d6f8d3 (11 minutes old) from quay.io for "quay.io/ redhattraining/httpd-parent"
     Red Hat Universal Base Image 8
The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.
    Tags: base rhel8
...output omitted...
--> Success
Build scheduled, use 'oc logs -f bc/hola' to track its progress. Application is not exposed. You can expose services to the outside world by
executing one or more of the commands below: 'oc expose svc/hola'
Run 'oc status' to view your app.
...output omitted...
```



## View the build logs


```
$ oc logs -f bc/hola
```



```
Cloning "https://github.com/youruser/xxxxx" ...
Commit: 823cd7a7476b5664cb267e5d0ac611de35df9f07 (Initial commit) Author: Your Name <youremail@example.com>
Date: Sun Jun 9 20:45:53 2019 -0400
Replaced Dockerfile FROM image quay.io/redhattraining/httpd-parent Caching blobs under "/var/cache/blobs".
Pulling image quay.io/redhattraining/httpd-parent@sha256:3e454fdac5
...output omitted...
Getting image source signatures Copying blob sha256:d02c3bd
...output omitted...
Writing manifest to image destination Storing signatures
STEP 1: FROM quay.io/redhattraining/httpd-parent@sha256:2833...86ff
STEP 2: COPY src/ ${DOCROOT}/
...output omitted...
Successfully pushed //image-registry.openshift-image-registry.svc:5000/... Push successful

```



## Verify that the application pod fails to start

The pod will be in the Error state, but if you wait too long, the pod will move to the CrashLoopBackOff state.


```
$  oc get pods
```



## Inspect the container's logs to see why the pod failed to start


```
$ oc logs hola-1-p75f5
```



```
AH00558: httpd: Could not reliably determine the server's fully qualified domain
name...
(13)Permission denied: AH00072: make_sock: could not bind to address [::]:80 (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:80 no listening sockets available, shutting down
AH00015: Unable to open logs
```



## Delete all the resources for next step


```
$ oc delete all -l app=hola
```



```
pod "hola-2-gbx6f" deleted
replicationcontroller "hola-1" deleted
service "hola" deleted
deploymentconfig.apps.openshift.io "hola" deleted 
buildconfig.build.openshift.io "hola" deleted 
build.build.openshift.io "hola-1" deleted 
imagestream.image.openshift.io "httpd-parent" deleted 
imagestream.image.openshift.io "hola" deleted 
route.route.openshift.io "hola" deleted
```



## Change the Dockerfile for the child container to run on an OpenShift 


### 1. Edit the ~/sunlife-training/developer-src/container-build/Dockerfile 


```
$ vi ~/sunlife-training/developer-src/container-build/Dockerfile 
```



### 2. Override the EXPOSE instruction 


```
EXPOSE 8080
```



### 3. Include the io.openshift.expose-service 


```
LABEL io.openshift.expose-services="8080:http"
```



### 4. Update the list of labels to include the io.k8s.description, io.k8s.display- name, and io.openshift.tags labels 


```
LABEL io.k8s.description="A basic Apache HTTP Server child image, uses ONBUILD" \
            io.k8s.display-name="Apache HTTP Server" \
            io.openshift.expose-services="8080:http" \
            io.openshift.tags="apache, httpd"
```



### 5. You need to run the web server on an unprivileged port 


```
RUN sed -i "s/Listen 80/Listen 8080/g" /etc/httpd/conf/httpd.conf
```



### 6. Change the group ID and permissions of the folders 


```
 RUN chgrp -R 0 /var/log/httpd /var/run/httpd && \
          chmod -R g=u /var/log/httpd /var/run/httpd
```



### 7. Add a USER instruction for an unprivileged user.


```
USER 1001
```



### 8. Save and Commit 


```
$ cd container-build
$ git commit -a -m "Changed Dockerfile to enable running as a random uid on OpenShift"
$ git push
$ cd ~
```



## Rebuild and redeploy the application


### 1. Re-create the application using the new Dockerfile


```
$ oc new-app --name hola  https://github.com/<urname>/sunlife-training --context-dir developer-scr/container-build

```



```
--> Found Docker image 1d6f8d3 (11 minutes old) from quay.io for "quay.io/ redhattraining/httpd-parent"
     Red Hat Universal Base Image 8
The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.
    Tags: base rhel8
...output omitted...
--> Success
Build scheduled, use 'oc logs -f bc/hola' to track its progress. Application is not exposed. You can expose services to the outside world by
executing one or more of the commands below: 'oc expose svc/hola'
Run 'oc status' to view your app.
...output omitted...
```



### 2. Wait until the build finishes and the pod is ready and running


```
$ oc get pods
```



## Create an OpenShift route to expose the application to external access


```
$ oc expose svc/hola
```



```
route.route.openshift.io/hola exposed
```



## Obtain the route URL using the oc get route command


```
$ oc get route
```





## Test the application using the route URL you obtained in the previous step


```
$ curl http://<the route>
```



```
...output omitted...
  Hello from the Apache child container!
...output omitted...
```



## Clean up


```
$ oc delete project <urname>-container-build
```



