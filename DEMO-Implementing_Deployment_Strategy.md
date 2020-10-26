<!-- Copy and paste the converted output. -->



# Implementing a Deployment Strategy


## Create a new project, and deploy an application based on the rhscl/mysql-57-rhel7 


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



### 2. Create a new project &lt;urname>--strategy


```
$ oc new-project sunny-strategy
```



```
Now using project "sunny-strategy" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Use the oc new-app command to create a new application called mysql


```
$ oc new-app --as-deployment-config --name mysql  -e MYSQL_USER=test -e MYSQL_PASSWORD=redhat -e MYSQL_DATABASE=testdb  --docker-image registry.access.redhat.com/rhscl/mysql-57-rhel7 

```



```
--> Found Docker image ... "registry.access.redhat.com/rhscl/mysql-57-rhel7"
...output omitted...
--> Creating resources ...
...output omitted...
--> Success
...output omitted...
```



### 4. Wait until the MySQL pod is deployed


```
$ oc get pod

```



## Patch the deployment configuration to change the default deployment strategy to Recreate


### 1. Verify that the default deployment strategy for the MySQL application is Rolling


```
$ oc get dc/mysql -o jsonpath='{.spec.strategy.type}{"\n"}'
```



```
Rolling
```



### 2. In the following steps, you will make several changes to the deployment configuration


```
$ oc set triggers dc/mysql --from-config --remove
```



```
deploymentconfig.apps.openshift.io/mysql triggers updatedl
```



### 3. Change the default deployment strategy


```
$ oc patch dc/mysql --patch  '{"spec":{"strategy":{"type":"Recreate"}}}'
```



### 4. Remove the rollingParams attribute from the deployment configuration


```
$ oc patch dc/mysql --type=json  -p='[{"op":"remove", "path": "/spec/strategy/rollingParams"}]'
```



```
deploymentconfig.apps.openshift.io/mysql patched
```



## Add a post life-cycle hook to initialize data for the MySQL database


### 1. Check out the SQL to be used at github (https://raw.githubusercontent.com/suchan-redhat/sunlife-training/main/developer-src/release/user.sql)


```
CREATE TABLE IF NOT EXISTS users (
    user_id int(10) unsigned NOT NULL AUTO_INCREMENT,
    name varchar(100) NOT NULL,
    email varchar(100)  NOT NULL,
    PRIMARY KEY (user_id)) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into users(name,email) values ('user1', 'user1@example.com');
insert into users(name,email) values ('user2', 'user2@example.com');
insert into users(name,email) values ('user3', 'user3@example.com');
```



### 2. Review below shell script (https://raw.githubusercontent.com/suchan-redhat/sunlife-training/main/developer-src/release/import.sh)


```
#!/bin/bash

# get timer and retries from env

if [ "$HOOK_RETRIES" = "" ]; then
  HOOK_RETRIES=0
fi
if [ "$HOOK_SLEEP" = "" ]; then
  HOOK_SLEEP=2
fi

cd /tmp

echo 'Downloading SQL script that initializes the database...'
curl -s -L -O https://raw.githubusercontent.com/suchan-redhat/sunlife-training/main/developer-src/release/user.sql

echo "Trying $HOOK_RETRIES times, sleeping $HOOK_SLEEP sec between tries:"
while [ "$HOOK_RETRIES" != 0 ]; do

  echo -n 'Checking if MySQL is up...'
  if mysqlshow -h$MYSQL_SERVICE_HOST -u$MYSQL_USER -p$MYSQL_PASSWORD -P3306 $MYSQL_DATABASE &>/dev/null
  then
    echo 'Database is up'
    break
  else
    echo 'Database is down'

    # Sleep to wait for the MySQL pod to be ready
    sleep $HOOK_SLEEP
  fi
  
  let HOOK_RETRIES=HOOK_RETRIES-1
done

if [ "$HOOK_RETRIES" = 0 ]; then
  echo 'Too many tries, giving up'
  exit 1
fi

# Run the SQL script
if mysql -h$MYSQL_SERVICE_HOST -u$MYSQL_USER -p$MYSQL_PASSWORD -P3306 $MYSQL_DATABASE < /tmp/users.sql
then
  echo 'Database initialized successfully'
else
  echo 'Failed to initialize database'
  exit 2
fi

```



### 3. Prepare below post-hook.sh


```
#!/bin/bash

oc patch dc/mysql --patch '{"spec":{"strategy":{"recreateParams":{"post":{"failurePolicy": "Abort","execNewPod":{"containerName":"mysql","command":["/bin/sh","-c","curl -L -s https://raw.githubusercontent.com/suchan-redhat/sunlife-training/main/developer-src/release/import.sh -o /tmp/import.sh&&chmod 755 /tmp/import.sh&&/tmp/import.sh"]}}}}}}'


```



### 4. Run post-hook.sh script


```
deploymentconfig.apps.openshift.io/mysql patched
```



## Verify the patched deployment configuration and roll out the new deployment configuration


### 1. Verify that the deployment strategy is now Recreate, and that there is a post life-cycle hook that executes the import.sh script


```
$ oc describe dc/mysql | grep -A 3 'Strategy:'
```



```
Strategy:	Recreate
  Post-deployment hook (pod type, failure policy: Abort):
    Container:	mysql
    Command:	/bin/sh -c curl -L -s ...
```



### 2. Force a new deployment to test changes to the strategy and the new post life-cycle hook


```
$ oc rollout latest dc/mysql
```



```
deploymentconfig.apps.openshift.io/mysql rolled out
```



### 3. Verify that a new MySQL pod comes up in a Running state


```
$ watch -n 2 oc get pods
```



```
NAME                READY     STATUS    RESTARTS   ...   NODE
mysql-2-deploy      1/1       Running   0          ...
mysql-2-hook-post   1/1       Running   0          ...
mysql-2-kbnpr       1/1       Running   0          ...  
```



### 4. After a few seconds, the post life-cycle hook pod and the second deployment pod fail


```
$ watch -n 2 oc get pods
```



```
NAME                READY     STATUS    RESTARTS   ...
mysql-1-rcw95       1/1       Running   0          ...
mysql-2-deploy      0/1       Error     0          ...
mysql-2-hook-post   0/1       Error     0          ...   
```



## Troubleshoot and fix the post life-cycle hook


### 1. Display the logs from the failed pod


```
$ oc logs mysql-2-hook-post
```



```
Downloading SQL script that initializes the database...
Trying 0 times, sleeping 2 sec between tries:
Too many tries, giving up
```



### 2. Notice the script incorrectly has a value of 0 for the HOOK_RETRIES variable


```
$ oc set env dc/mysql HOOK_RETRIES=5
```



```
deploymentconfig.apps.openshift.io/mysql updated
```



### 3. Start a third deployment to run the hook a second time, using the new values for the environment variables


```
$ oc rollout latest dc/mysql
```



```
deploymentconfig.apps.openshift.io/mysql rolled out
```



### 4. Wait until the new post life-cycle hook pod is in a status of Completed


```
$ watch -n 2 oc get pods
```



```
NAME                READY     STATUS    RESTARTS   ...
mysql-1-deploy      0/1     Completed   0          5m26s
mysql-2-deploy      0/1     Error       0          3m30s
mysql-2-hook-post   0/1     Error       0          3m8s
mysql-3-29jwt       1/1     Running     0          57s
mysql-3-deploy      0/1     Completed	0          79s
mysql-3-hook-post   0/1     Completed   0          48s
```



### 5. Open a new terminal to display the logs from the running post life-cycle hook pod


```
$ oc logs -f mysql-3-hook-post
```



```
Downloading SQL script that initializes the database...
Trying 5 times, sleeping 2 sec between tries:
Checking if MySQL is up...Database is up
mysql: [Warning] Using a password on the command line interface can be insecure.
Database initialized successfully
```



### 6. After a few seconds, the new database pod is the one created using the latest changes to the deployment configuration:


```
NAME                READY     STATUS    RESTARTS   ...
mysql-1-deploy      0/1     Completed   0          7m46s
mysql-2-deploy      0/1     Error	0          5m50s
mysql-2-hook-post   0/1     Error	0          5m28s
mysql-3-29jwt       1/1     Running     0          3m17s
mysql-3-deploy      0/1     Completed   0          3m39s
mysql-3-hook-post   0/1     Completed   0          3m8s
```



## Verify that the new MySQL database pod contains the data from the SQL file


### 1. Open a shell session to the MySQL container pod


```
$ oc get pods
```



```
NAME            READY     STATUS    RESTARTS   AGE
...output omitted...
mysql-3-3p4m1   1/1       Running   0          8m
```



```
$ oc rsh mysql-3-3p4m1
```



```
sh-4.2$
```



### 2. Verify that the users table has been created and populated with data from the SQL file


```
$ mysql -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE
```



```
...output omitted...
Server version: 5.7.16 MySQL Community Server (GPL)
...output omitted...
mysql> select * from users;
+---------+-------+-------------------+
| user_id | name  | email             |
+---------+-------+-------------------+
|       1 | user1 | user1@example.com |
|       2 | user2 | user2@example.com |
|       3 | user3 | user3@example.com |
+---------+-------+-------------------+
3 rows in set (0.00 sec)
```



### 3. Exit the MySQL session and the container shell


```
$ mysql> exit
Bye
sh-4.2$ exit
exit
$
```



## Clean up


```
$ oc delete project  <urname>-strategy
```


