<!-- Copy and paste the converted output. -->



# Managing Sensitive Information With Secrets


## Login to the OpenShift cluster and create the authorization-secrets-&lt;urname> project.


### 1. Login to OCP


```
$ oc login -u opentlc-mgr -p 'r3dh4t1!' https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443
```



```
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Login successful.

You have access to 273 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```



### 2. Create the authorization-secrets-&lt;urname> project.


```
$ oc new-project authorization-secrets-sunny
```



```
Now using project "authorization-secrets-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Create a secret with the credentials and connection information to access a MySQL database. 


```
$ oc create secret generic mysql --from-literal user=myuser --from-literal password=redhat123   --from-literal database=test_secrets --from-literal hostname=mysql
```



```
secret/mysql created
```



## Deploy a database and add the secret for user and database configuration.


### 1. Try to deploy an ephemeral database server. This should fail because the MySQL image needs environment variables for its initial configuration. The values for these variables cannot be assigned using secrets because the oc new-app command does not support the use of this type of secrets.


```
$ oc new-app --name mysql  --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7:5.7-47
```



```
warning: Cannot find git. Ensure that it is installed and in your path. Git is required to work with git repositories.
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



### 2. Run the oc get pods command with the -w option to retrieve the status of the deployment in real time. Notice how the database pod is in a failed state. Type Ctrl+C to exit the command.


```
$ oc get pods -w
```



```
NAME             READY   STATUS             RESTARTS   AGE
mysql-1-deploy   1/1     Running            0          6m52s
mysql-1-kdcpf    0/1     CrashLoopBackOff   6          6m49s
```



### 3. Setting the mysql secret as an environment variable to the deployment configuration triggers a new build of the application.

Use the mysql secret to initialize environment variables on the mysql deployment configuration. The deployment needs the MYSQL_USER, MYSQL_PASSWORD, and MYSQL_DATABASE environment variables for a successful initialization. The secret has the user, password, and database keys that can be assigned to the deployment configuration as environment variables, adding the prefix MYSQL_. 


```
$ oc set env dc/mysql --prefix MYSQL_  --from secret/mysql
```



```
deploymentconfig.apps.openshift.io/mysql updated
```



### 4. Verify that the mysql application was deployed successfully after adding the secret.


```
$  oc get pods
```



```
NAME             READY   STATUS      RESTARTS   AGE
mysql-2-deploy   0/1     Completed   0          74s
mysql-2-slj6c    1/1     Running     0          71s
```



## Verify that the database now authenticates using the environment variables initialized from the mysql secret.


### 1. Open a remote shell session to the mysql pod in the Running state.


```
$ oc rsh $(oc get pods |grep Running | awk '{print $1}')
```



```
sh-4.2$
```



### 2. Start a MySQL session to verify that the environment variables initialized by the mysql secret were used to configure the mysql application.


```
$ mysql -u myuser --password=redhat123 test_secrets -e 'show databases;'
```



```
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| test_secrets       |
+--------------------+
```



### 3. Close the remote shell session to continue from your workstation.


```
sh-4.2$ exit
```



```
exit
```



## Create a new application that uses the MySQL database.


### 1. Create a new application using the redhattraining/famous-quotes image from Quay.io.


```
$ oc new-app --name quotes  --docker-image quay.io/redhattraining/famous-quotes:1.0
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



### 2. Verify the quotes application pod status. This should display an error because you have not added the environment variables it needs to connect to the database. This might take a while to display in the output.


```
$ oc get pods -w
```



```
NAME              READY   STATUS             RESTARTS   AGE
mysql-2-deploy    0/1     Completed          0          16m
mysql-2-slj6c     1/1     Running            0          16m
quotes-1-deploy   1/1     Running            0          4m37s
quotes-1-pghld    0/1     CrashLoopBackOff   5          4m34s
```



## Use the mysql secret to initialize the environment variables that the quotes application needs to connect to the database.

The quotes application requires the following environment variables: QUOTES_USER, QUOTES_PASSWORD, QUOTES_DATABASE, and QUOTES_HOSTNAME, which correspond

to the user, password, database, and hostname keys of the mysql secret. As in the MySQL application case, the secret can be added to the quotes application to initialize its environment variables with the prefix QUOTES_.


### 1. Use the mysql secret to initialize environment variables in the application so it can connect to the mysql database.


```
$ oc set env dc/quotes --prefix QUOTES_  --from secret/mysql
```



```
deploymentconfig.apps.openshift.io/quotes updated 
```



### 2. Wait until the quotes application is successfully redeployed. The older containers should be automatically terminated..


```
$ oc get pods
```



```
NAME              READY   STATUS      RESTARTS   AGE
mysql-2-deploy    0/1     Completed   0          31m
mysql-2-slj6c     1/1     Running     0          31m
quotes-2-79dzx    1/1     Running     0          13m
quotes-2-deploy   0/1     Completed   0          13m
```



## Verify that the quotes application environment variables were successfully set and that the application is working properly.


### 1. Verify that the application started correctly.


```
$ oc logs  $(oc get pods |grep Running |grep quotes| awk '{print $1}')
```



```
2020/10/14 02:15:03 Connecting to the database: myuser:redhat123@tcp(mysql:3306)/test_secrets
2020/10/14 02:15:03 Database connection OK
2020/10/14 02:15:03 Creating schema
2020/10/14 02:15:03 Adding quotes
2020/10/14 02:15:03 Adding quote: When words fail, music speaks.
- William Shakespeare
2020/10/14 02:15:03 Adding quote: Happines depends upon ourselves.
- Aristotle
2020/10/14 02:15:03 Adding quote: The secret of change is to focus all your energy not on fighting the old but on building the new.
- Socrates
2020/10/14 02:15:03 Adding quote: Nothing that glitters is gold.
- Mark Twain
2020/10/14 02:15:03 Adding quote: Imagination is more important than knowledge.
- Albert Einstein
2020/10/14 02:15:03 Adding quote: Hell, if I could explain it to the average person, it wouldn't have been worth the Nobel prize.
- Richard Feynman
2020/10/14 02:15:03 Adding quote: Young man, in mathematics you don't understand things. You just get used to them.
- John von Neumann
2020/10/14 02:15:03 Adding quote: Those who can imagine anything, can create the impossible.
- Alan Turing
2020/10/14 02:15:03 Database Setup Completed
2020/10/14 02:15:03 Starting Application
Services:
/
/random
/env
/status
```



### 2. Expose the application service so that it can be accessed from out side the cluster.


```
$ oc expose service quotes
```



```
route.route.openshift.io/quotes exposed
```



### 3. Verify the application host name.


```
$ oc get route quotes
```



```
NAME     HOST/PORT                                                                                           PATH   SERVICES   PORT       TERMINATION   WILDCARD
quotes   quotes-authorization-secrets-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com          quotes     8000-tcp                 None
```



### 4. Verify that the variables are being properly set in the application by accessing the env REST entry point.


```
$ curl http://quotes-authorization-secrets-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/env
```



```
<html>
	<head>
        <title>Quotes</title>
    </head>
    <body>
       
        <h1>Quote List</h1>
        
            <h3>Database Environment Variables</h3>
            <ul>
                <li>QUOTES_USER: myuser </li>
                <li>QUOTES_PASSWORD: redhat123 </li>
                <li>QUOTES_DATABASE: test_secrets</li>
                <li>QUOTES_HOST: mysql</li>
            </ul>
        
    </body>
</html>
```



### 5. Access the application status REST API entrypoint to test the database connection


```
$ curl http://quotes-authorization-secrets-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/status
```



```
Database connection OK
```



### 6. Test application functionality by accessing the random RESTAPI entrypoint


```
$ curl http://quotes-authorization-secrets-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/random
```



```
5: Imagination is more important than knowledge.
- Albert Einstein
```



## Remove the authorization-secrets-&lt;urname> project


```
$ oc delete project authorization-secrets-sunny
```



```
project.project.openshift.io "authorization-secrets-sunny" deleted
```


