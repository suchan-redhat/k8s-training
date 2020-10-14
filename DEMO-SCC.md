<!-- Copy and paste the converted output. -->



# Controlling Application Permissions with Security Context Constraints (SCC)


## Login to the OpenShift cluster and create the authorization-scc-&lt;urname> project.


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



### 2. Create the authorization-scc-&lt;urname> project.


```
$ oc new-project authorization-scc-sunny
```



```
Now using project "authorization-scc-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



## Deploy a gitlab/gitlab-ce:8.4.3-ce.0 application and verify that it fails because the container image needs root privileges.


### 1. Deploy a gitlab/gitlab-ce:8.4.3-ce.0 application.


```
$ oc new-app --name gitlab gitlab/gitlab-ce:8.4.3-ce.0
```



```
--> Found container image a26371b (4 years old) from Docker Hub for "gitlab/gitlab-ce:8.4.3-ce.0"

    * An image stream tag will be created as "gitlab:8.4.3-ce.0" that will track this image
    * This image will be deployed in deployment config "gitlab"
    * Ports 22/tcp, 443/tcp, 80/tcp will be load balanced by service "gitlab"
      * Other containers can access this service through the hostname "gitlab"
    * This image declares volumes and will default to use non-persistent, host-local storage.
      You can add persistent volumes later by running 'oc set volume dc/gitlab --add ...'
    * WARNING: Image "gitlab/gitlab-ce:8.4.3-ce.0" runs as the 'root' user which may not be permitted by your cluster administrator

--> Creating resources ...
    imagestream.image.openshift.io "gitlab" created
    deploymentconfig.apps.openshift.io "gitlab" created
    service "gitlab" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/gitlab' 
    Run 'oc status' to view your app.

```



### 2. Determine if the application was successfully deployed. It should give an error because this image needs root privileges to properly deploy.


```
$ oc get pods
```



```
NAME              READY   STATUS      RESTARTS   AGE
gitlab-1-deploy   0/1     Completed   0          63s
gitlab-1-mg55k    0/1     Error       1          60s
```



### 3. Review the application logs to confirm that the failure is caused by insufficient privileges.


```
$ oc logs  gitlab-1-mg55k
```



```
...output omitted... 

================================================================================
Recipe Compile Error in /opt/gitlab/embedded/cookbooks/cache/cookbooks/gitlab/recipes/default.rb
================================================================================

Chef::Exceptions::InsufficientPermissions
-----------------------------------------
directory[/etc/gitlab] (gitlab::default line 26) had an error: Chef::Exceptions::InsufficientPermissions: Cannot create directory[/etc/gitlab] at /etc/gitlab due to insufficient permissions

...output omitted... 
```



## Create a new service account and add the anyuid SCC to it.


### 1. Create a service account named gitlab-sa


```
$ oc create sa gitlab-sa
```



```
serviceaccount/gitlab-sa created
```



### 2. Login as the admin user.


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



### 3. Assign the anyuid SCC to the gitlab-sa service account.


```
$ oc adm policy add-scc-to-user anyuid -z gitlab-sa
```



```
securitycontextconstraints.security.openshift.io/anyuid added to: ["system:serviceaccount:authorization-scc-sunny:gitlab-sa"]
```



## Modify the gitlab application so that it uses the newly created service account. Verify that the new deployment is successful


### 1. Login as the developer user.


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



### 2. Assign the gitlab-sa service account to the gitlab deployment.


```
$ oc set serviceaccount deploymentconfig gitlab gitlab-sa
```



```
deploymentconfig.apps.openshift.io/gitlab serviceaccount updated.
```



### 3. Verify that the gitlab redeployment was successful. You may need to run the oc get pods command multiple times until you see a running application pod


```
$ oc get pod
```



```
NAME              READY   STATUS      RESTARTS   AGE
gitlab-1-deploy   0/1     Completed   0          13m
gitlab-2-5b2qx    1/1     Running     0          95s
gitlab-2-deploy   0/1     Completed   0          98s
```



## Verify that the gitlab application is properly working


### 1. Expose the gitlab application


```
$ oc expose service gitlab --port 80
```



```
route.route.openshift.io/gitlab exposed
```



### 2. Get the exposed route


```
$ oc get route gitlab
```



```
NAME     HOST/PORT                                                                                       PATH   SERVICES   PORT   TERMINATION   WILDCARD
gitlab   gitlab-authorization-scc-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com          gitlab     80                   None
```



### 2. Verify that the gitlab application is answering HTTP queries.


```
$ curl http://gitlab-authorization-scc-sunny.apps.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com/users/sign_in | grep title
```



```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  9160    0  9160    0     0  14114      0 --:--:-- --:--:-- --:--:-- 14114
<meta content='Sign in' property='og:title'>
<meta content='Sign in' property='twitter:title'>
<title>Sign in · GitLab</title>
```



## Delete the authorization-scc-&lt;urname> project


```
$ oc delete project authorization-scc-sunny
```



```
project.project.openshift.io "authorization-scc-sunny" deleted
```


