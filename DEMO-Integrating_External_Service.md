<!-- Copy and paste the converted output. -->



# Integrating an External Service


## Create an application that use mysql database


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



### 2. Create a new project &lt;urname>-external-service


```
$ oc new-project sunny-external-service
```



```
Now using project "sunny-external-service" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Create an application using the oc new app command


```
$ oc new-app --name quotes  --docker-image quay.io/redhattraining/famous-quotes:1.0 -e QUOTES_USER=SWEmsNvNkQ  -e QUOTES_PASSWORD=vAPlMlJXH5 -e QUOTES_DATABASE=SWEmsNvNkQ -e QUOTES_HOSTNAME=mysql
```



```
warning: Cannot find git. Ensure that it is installed and in your path. Git is required to work with git repositories.
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



### 4. Verify that the application didn’t run


```
$ oc get pods
```



```
NAME              READY   STATUS    RESTARTS   AGE
quotes-1-4l4gn    0/1     Error     0          3s
quotes-1-deploy   1/1     Running   0          5s
```



### 5. Check the error 


```
$ oc logs quotes-1-4l4gn --previous
```



```
2020/10/27 13:20:08 Connecting to the database: SWEmsNvNkQ:vAPlMlJXH5@tcp(mysql:3306)/SWEmsNvNkQ
2020/10/27 13:20:08 Database connection error: dial tcp: lookup mysql on 172.30.0.10:53: no such host
```



## Inspect the external database


### 1. Connect to the external database by local mysql command


```
$ mysql -u SWEmsNvNkQ -pvAPlMlJXH5 -h remotemysql.com
```



```
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 51639401
Server version: 8.0.13-4 Percona Server (GPL), Release '4', Revision 'f0a32b8'

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>

```



## Create an OpenShift service that connects to the external database instance


### 1. Use the oc create svc command to create a service based on an external name


```
$ oc create svc externalname mysql  --external-name remotemysql.com
```



```
service/mysql created
```



### 2. Verify that the tododb service exists and shows an external IP, but no cluster IP


```
$ oc get svc
```



```
NAME     TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)    AGE
mysql    ExternalName   <none>         remotemysql.com   <none>     2m32s
quotes   ClusterIP      172.30.61.89   <none>            8000/TCP   79m
```



### 3. Verify that the application now can get data from the external database by rollout another deployment


```
$ oc rollout latest dc/quotes
```



### 4. Wait for the pod 


```
$ oc get pods
```



```
NAME              READY   STATUS      RESTARTS   AGE
quotes-1-deploy   0/1     Error       0          82m
quotes-2-6smx8    1/1     Running     0          47s
quotes-2-deploy   0/1     Completed   0          49s
```



### 5. Expose the route


```
$ oc expose svc quotes
```



```
route.route.openshift.io/quotes exposed
```



### 6. Test the application using the route URL you obtained in the previous step


```
$ curl  http://<the route>/random

```



```
1: When words fail, music speaks.
- William Shakespeare
```



## Clean up


```
$ oc delete project  <urname>-external-service
```


