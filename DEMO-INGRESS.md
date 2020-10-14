<!-- Copy and paste the converted output. -->



# Controlling Cluster Network Ingress


## Login to the OpenShift cluster and create the network-ingres-&lt;urname> project.


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



### 2. Create the network-ingres-&lt;urname> project.


```
$ oc new-project network-ingres-sunny
```



```
Now using project "network-ingres-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Create testing application


### 1. Deploy a MySQL application 


```
$ oc new-app --name mysql  --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47 -e MYSQL_USER=myuser -e MYSQL_PASSWORD=password -e MYSQL_DATABASE=mydb
```



```
--> Found container image 77d20f2 (14 months old) from registry.access.redhat.com for "registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47"

    MySQL 5.7 
    --------- 
    MySQL is a multi-user, multi-threaded SQL database server. The container image provides a containerized packaging of the MySQL mysqld daemon and client application. The mysqld server daemon accepts connections from clients and provides access to content from MySQL databases on behalf of the clients.

    Tags: database, mysql, mysql57, rh-mysql57

    * An image stream tag will be created as "mysql:5.7-47" that will track this image
    * This image will be deployed in deployment config "mysql"
    * Port 3306/tcp will be load balanced by service "mysql"
      * Other containers can access this service through the hostname "mysql"

--> Creating resources ...
    imagestream.image.openshift.io "mysql" created
    deploymentconfig.apps.openshift.io "mysql" created
    service "mysql" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/mysql' 
    Run 'oc status' to view your app.
```



### 2. Create the quotes app


```
oc new-app --name quotes  --docker-image quay.io/redhattraining/famous-quotes:1.0 -e QUOTES_USER=myuser  -e QUOTES_PASSWORD=password -e QUOTES_DATABASE=mydb -e QUOTES_HOSTNAME=mysql
```



```
--> Found container image ff11463 (10 months old) from quay.io for "quay.io/redhattraining/famous-quotes:1.0"

    Quotes 1.0 
    ---------- 
    Famous Quotes is a PoC application for Go and MySQL

    Tags: poc, mysql, golang

    * An image stream tag will be created as "quotes:1.0" that will track this image
    * This image will be deployed in deployment config "quotes"
    * Port 8000/tcp will be load balanced by service "quotes"
      * Other containers can access this service through the hostname "quotes"
    * WARNING: Image "quay.io/redhattraining/famous-quotes:1.0" runs as the 'root' user which may not be permitted by your cluster administrator

--> Creating resources ...
    imagestream.image.openshift.io "quotes" created
    deploymentconfig.apps.openshift.io "quotes" created
    service "quotes" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/quotes' 
    Run 'oc status' to view your app.
```



### 3. Wait few minutes and run the oc get pod


```
$ oc get status
```



```
In project network-ingres-sunny on server https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443

svc/mysql - 172.30.184.23:3306
  dc/mysql deploys istag/mysql:5.7-47 
    deployment #1 deployed 48 seconds ago - 1 pod

svc/quotes - 172.30.60.161:8000
  dc/quotes deploys istag/quotes:1.0 
    deployment #1 deployed 28 seconds ago - 1 pod


4 infos identified, use 'oc status --suggest' to see details
```



### 4. Expose the service


```
$ oc expose svc quotes
```



```
route.route.openshift.io/quotes exposed
```



### 5. Get the name of the route


```
$ oc get routes
```



```
NAME     HOST/PORT                                                                                    PATH   SERVICES   PORT       TERMINATION   WILDCARD
quotes   quotes-network-ingres-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com          quotes     8000-tcp                 None
```



### 6. Try to access the route from your browser


```
$ curl http://quotes-network-ingres-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/random
```



```
1: When words fail, music speaks.
- William Shakespeare
```



## Create a secure edge route


### 1. Use oc create route to create the edge route using https as prefix to the previous route exposed


```
$ oc create route edge quotes-https  --service quotes --hostname https-quotes-network-ingres-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com

```



```
route.route.openshift.io/quotes-https created
```



### 2. Use curl to test the route


```
$ curl https://https-quotes-network-ingres-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/random
```



```
6: Hell, if I could explain it to the average person, it wouldn't have been worth the Nobel prize.
- Richard Feynman
```


