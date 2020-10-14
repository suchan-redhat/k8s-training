<!-- Copy and paste the converted output. -->



# Troubleshooting OpenShift Software- defined Networking


## Login to the OpenShift cluster and create the network-sdn-&lt;urname> project.


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



### 2. Create the network-sdn-&lt;urname> project.


```
$ oc new-project network-sdn-sunny
```



```
Now using project "network-sdn-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Create MySQL Server and restore data


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



### 2. Restore Database

Create a file with below content as dbdump.sql


```
-- MySQL dump 10.13  Distrib 5.7.24, for Linux (x86_64)
--
-- Host: localhost    Database: mydb
-- ------------------------------------------------------
-- Server version	5.7.24

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `quotes`
--

DROP TABLE IF EXISTS `quotes`;
/*!40101 SET @saved_cs_client     = @@character_set_client */;
/*!40101 SET character_set_client = utf8 */;
CREATE TABLE `quotes` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `message` text,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=latin1;
/*!40101 SET character_set_client = @saved_cs_client */;

--
-- Dumping data for table `quotes`
--

LOCK TABLES `quotes` WRITE;
/*!40000 ALTER TABLE `quotes` DISABLE KEYS */;
INSERT INTO `quotes` VALUES (1,'Sunlife Training\n- Sunny Chan\n');
/*!40000 ALTER TABLE `quotes` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;

/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;

-- Dump completed on 2020-10-14 12:29:04

```


Copy the file into the database pod


```
$ oc cp dbdump.sql mysql-1-75wfm:/tmp/dbdump.sql
```


Login to the pod 


```
$  oc rsh mysql-1-75wfm bash
```



```
bash-4.2$
```


perform database restore


```
bash-4.2$ mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD $MYSQL_DATABASE < /tmp/dbdump.sql
```



```
mysql: [Warning] Using a password on the command line interface can be insecure.
```



### 3. Ensure Quotes table exist and its data


```
bash-4.2$ mysql -u $MYSQL_USER  -p$MYSQL_PASSWORD $MYSQL_DATABASE -e "select * from quotes;"
```



```
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+--------------------------------+
| id | message                        |
+----+--------------------------------+
|  1 | Sunlife Training
- Sunny Chan
 |
+----+--------------------------------+
```



## Create a quote frontend app


### 1. Create the quotes app


```
$ cat <<EOF | oc create -f -
apiVersion: v1
items:
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    annotations:
      openshift.io/generated-by: OpenShiftNewApp
    creationTimestamp: null
    labels:
      app: quotes
      app.kubernetes.io/component: quotes
      app.kubernetes.io/instance: quotes
    name: quotes
  spec:
    replicas: 1
    selector:
      deploymentconfig: quotes
    strategy:
      resources: {}
    template:
      metadata:
        annotations:
          openshift.io/generated-by: OpenShiftNewApp
        creationTimestamp: null
        labels:
          deploymentconfig: quotes
      spec:
        containers:
        - env:
          - name: QUOTES_DATABASE
            value: mydb
          - name: QUOTES_HOSTNAME
            value: mysql
          - name: QUOTES_PASSWORD
            value: password
          - name: QUOTES_USER
            value: myuser
          image: quay.io/redhattraining/famous-quotes:1.0
          name: quotes
          ports:
          - containerPort: 8000
            protocol: TCP
          resources: {}
    test: false
    triggers:
    - type: ConfigChange
    - imageChangeParams:
        automatic: true
        containerNames:
        - quotes
        from:
          kind: ImageStreamTag
          name: quotes:1.0
      type: ImageChange
  status:
    availableReplicas: 0
    latestVersion: 0
    observedGeneration: 0
    replicas: 0
    unavailableReplicas: 0
    updatedReplicas: 0
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      openshift.io/generated-by: OpenShiftNewApp
    creationTimestamp: null
    labels:
      app: quotes
      app.kubernetes.io/component: quotes
      app.kubernetes.io/instance: quotes
    name: quotes
  spec:
    ports:
    - name: 8000-tcp
      port: 8000
      protocol: TCP
      targetPort: 8000
  status:
    loadBalancer: {}
kind: List
metadata: {}
EOF
```



```
deploymentconfig.apps.openshift.io/quotes created
service/quotes created
```



### 2. Wait few minutes and run the oc get pod


```
$ oc get pod
```



```
NAME              READY   STATUS      RESTARTS   AGE
mysql-1-75wfm     1/1     Running     0          29m
mysql-1-deploy    0/1     Completed   0          29m
quotes-1-deploy   0/1     Completed   0          27m
quotes-3-deploy   0/1     Completed   0          62s
quotes-3-m9g9m    1/1     Running     0          58s
```



## Create a route to access the quotes service and access the application


### 1. Expose the service


```
$ oc expose svc quotes
```



```
route.route.openshift.io/quotes exposed
```



### 2. List the routes in the project.


```
$ oc get routes
```



```
NAME     HOST/PORT                                                                                 PATH   SERVICES   PORT       TERMINATION   WILDCARD
quotes   quotes-network-sdn-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com          quotes     8000-tcp                 None
```



### 3. Try to access the route from your browser


```
$ curl quotes-network-sdn-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com
```



```

… output omitted …

    <div>
      <h1>Application is not available</h1>
      <p>The application is currently not serving requests at this endpoint. It may not have been started or is still starting.</p>

      <div class="alert alert-info">
        <p class="info">
          Possible reasons you are seeing this page:
        </p>
        <ul>
          <li>
            <strong>The host doesn't exist.</strong>
            Make sure the hostname was typed correctly and that a route matching this hostname exists.
          </li>
          <li>
            <strong>The host exists, but doesn't have a matching path.</strong>
            Check if the URL path was typed correctly and that the route was created using the desired path.
          </li>
          <li>
            <strong>Route and path matches, but all pods are down.</strong>
            Make sure that the resources exposed by this route (pods, services, deployment configs, etc) have at least one pod running.
          </li>

… output omitted ...
```



### 4. Inspect the pod logs for errors. The output does not indicate any errors.


```
$ oc logs quotes-3-m9g9m
```



```
2020/10/14 12:52:54 Connecting to the database: myuser:password@tcp(mysql:3306)/mydb
2020/10/14 12:52:54 Database connection OK
2020/10/14 12:52:54 Database already setup, ignoring
2020/10/14 12:52:54 Starting Application
Services:
/
/random
/env
/status
```



## Run oc debug to create a carbon copy of an existing pod in the quotes deployment. You use this pod to check connectivity to the database.


### 1. Retrieve the database service IP


```
$ oc get service/mysql  -o jsonpath="{.spec.clusterIP}{'\n'}"
```



```
172.30.93.109
```



### 2. Run the oc debug command against the quotes deployment, which runs the web application pod.


```
$ oc debug -t dc/quotes
```



```
Starting pod/quotes-debug, command was: ./famous-quotes
Pod IP: 10.131.0.139
If you don't see a command prompt, try pressing enter.

```



### 3. Use curl to connect to the database over the port 3306


```
$ curl -v telnet://172.30.93.109:3306
```



```
* Expire in 0 ms for 6 (transfer 0x5606da933f50)
*   Trying 172.30.93.109...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x5606da933f50)
* Connected to 172.30.93.109 (172.30.93.109) port 3306 (#0)
Warning: Binary output can mess up your terminal. Use "--output -" to tell 
Warning: curl to output it to your terminal anyway, or consider "--output 
Warning: <FILE>" to save to a file.
* Failed writing body (0 != 25)
* Closing connection 0
$ exit

Removing debug pod ...
```


The output indicates that the database is up and running, and that it is accessible from the quotes deployment.


## Ensure that the network connectivity inside the cluster is operational


### 1. Before creating a debug pod, retrieve IP address of the quotes service


```
$ oc get service/quotes -o jsonpath="{.spec.clusterIP}{'\n'}"
```



```
172.30.18.116
```



### 2. Run the oc debug command to create a container for troubleshooting based on the mysql deployment. You must override the container image because the MySQL Server image does not provide the curl command.


```
$ oc debug -t dc/mysql  --image registry.access.redhat.com/ubi8/ubi:8.0
```



```
Starting pod/mysql-debug, command was: container-entrypoint run-mysqld
Pod IP: 10.129.4.97
If you don't see a command prompt, try pressing enter.
sh-4.4$ 

```



### 2. Use curl to connect to the quotes application over the port 8080


```
$ curl http://gitlab-authorization-scc-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/users/sign_in | grep title
```



```
 sh-4.4$ curl -m 10 -v http://172.30.93.109:8000
* Rebuilt URL to: http://172.30.93.109:8000/
*   Trying 172.30.93.109...
* TCP_NODELAY set
* connect to 172.30.93.109 port 8000 failed: No route to host
* Failed to connect to 172.30.93.109 port 8000: No route to host
* Closing connection 0
curl: (7) Failed to connect to 172.30.93.109 port 8000: No route to host

```



### 3. Exit the debug pod


```
sh-4.4$ exit
```



```
Removing debug pod ... 
```



## Connect to the quotes pod through its private IP


### 1. Retrieve the IP of the quotes pod


```
$ oc get pods -o wide -l deploymentconfig=quotes
```



```
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE                                              NOMINATED NODE   READINESS GATES
quotes-3-m9g9m   1/1     Running   0          24m   10.130.4.65   ip-10-0-166-110.ap-southeast-1.compute.internal   <none>           <none>
```



### 2. Create a debug pod from the mysql deployment


```
$ oc debug -t dc/mysql --image registry.access.redhat.com/ubi8/ubi:8.0
```



```
Starting pod/mysql-debug, command was: container-entrypoint run-mysqld
Pod IP: 10.129.4.98
If you don't see a command prompt, try pressing enter.
sh-4.4$
```



### 3. Run the curl command against quotes pod on port 8000


```
$ curl -v http://10.130.4.65:8000/status
```



```
sh-4.4$ curl -v http://10.130.4.65:8000/status 
*   Trying 10.130.4.65...
* TCP_NODELAY set
* Connected to 10.130.4.65 (10.130.4.65) port 8000 (#0)
> GET /status HTTP/1.1
> Host: 10.130.4.65:8000
> User-Agent: curl/7.61.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Wed, 14 Oct 2020 13:19:46 GMT
< Content-Length: 23
< Content-Type: text/plain; charset=utf-8
< 
Database connection OK
* Connection #0 to host 10.130.4.65 left intact
```


Curl can access the application through the pod's private IP


### 4. Exit the debug pod


```
sh-4.4$ exit
```



```
Removing debug pod ... 
```



## Review the quotes service


### 1. List the service in the project and make sure quotes service exist


```
$ oc get svc
```



```
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
mysql    ClusterIP   172.30.93.109   <none>        3306/TCP   57m
quotes   ClusterIP   172.30.18.116   <none>        8000/TCP   55m
```



### 2. Review the configuration and status of the quotes service


```
$ oc describe svc/quotes
```



```
Name:              quotes
Namespace:         network-sdn-sunny
Labels:            app=quotes
                   app.kubernetes.io/component=quotes
                   app.kubernetes.io/instance=quotes
Annotations:       openshift.io/generated-by: OpenShiftNewApp
Selector:          <none>
Type:              ClusterIP
IP:                172.30.18.116
Port:              8000-tcp  8000/TCP
TargetPort:        8000/TCP
Endpoints:         10.131.0.138:8000
Session Affinity:  None
Events:            <none>
```


Indicates that the service has no endpoint, so it is not able to forward incoming traffic to the application.


### 3. Retrieve the labels of the quotes deployment config


```
$ oc get pod --show-labels
```



```
NAME              READY   STATUS      RESTARTS   AGE   LABELS
mysql-1-75wfm     1/1     Running     0          63m   deployment=mysql-1,deploymentconfig=mysql
mysql-1-deploy    0/1     Completed   0          63m   openshift.io/deployer-pod-for.name=mysql-1
quotes-1-deploy   0/1     Completed   0          61m   openshift.io/deployer-pod-for.name=quotes-1
quotes-3-deploy   0/1     Completed   0          34m   openshift.io/deployer-pod-for.name=quotes-3
quotes-3-m9g9m    1/1     Running     0          34m   deployment=quotes-3,deploymentconfig=quotes
```



## Update the quotes service and access the application


### 1. Run oc edit to edit the quotes service 


```
$ oc edit svc quotes
```



```
...output omitted...
  - name: 8000-tcp
    port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    deploymentconfig: quotes
  sessionAffinity: None
...output omitted...
```



### 2. Review the service and make sure it has an endpoint


```
$ oc describe svc/quotes
```



```
Name:              quotes
Namespace:         network-sdn-sunny
Labels:            app=quotes
                   app.kubernetes.io/component=quotes
                   app.kubernetes.io/instance=quotes
Annotations:       openshift.io/generated-by: OpenShiftNewApp
Selector:          deploymentconfig=quotes
Type:              ClusterIP
IP:                172.30.18.116
Port:              8000-tcp  8000/TCP
TargetPort:        8000/TCP
Endpoints:         10.130.4.65:8000
Session Affinity:  None
Events:            <none>
```



### 3. Test the application from your desktop


```
$ curl http://quotes-network-sdn-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/random
```



```
1: Sunlife Training
- Sunny Chan
```



## Clean up the project


```
$ oc delete project network-sdn-sunny
```



```
project.project.openshift.io "network-sdn-sunny" deleted
```


