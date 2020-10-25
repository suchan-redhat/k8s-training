<!-- Copy and paste the converted output. -->



# Customizing S2I Builds


## Explore the S2I scripts packaged in the rhscl/httpd-24-rhel7 builder image


### 1. run the rhscl/httpd-24-rhel7 image from a terminal window


```
$ sudo podman run --name test -it rhscl/httpd-24-rhel7 bash
```



```
$ Trying to pull registry.access.redhat.com/rhscl/httpd-24-rhel7:latest...Getting image source signatures
...output omitted...
bash-4.2$
```



### 2. Inspect the S2I scripts packaged in the builder image


```
bash-4.2$ cat /usr/libexec/s2i/assemble
...output omitted...
bash-4.2$ cat /usr/libexec/s2i/run
...output omitted...
bash-4.2$ cat /usr/libexec/s2i/usage
...output omitted…
bash-4.2$ exit
```



## Review the application source code with the custom S2I scripts


### 1. Preparing your Repo


```
$ git clone https://github.com/<ur own repo>/sunlife-training.git
```



### 2. Inspect the ~/sunlife-training/developer-src/s2i-scripts/index.html file.


```
$ cat ~/sunlife-training/developer-src/s2i-scripts/index.html
```



```
Hello Class! Sunlife Developer rocks!!!
```



### 3. Inspect the custom S2I scripts are in the ~/sunlife-training/developer-src/s2i-scripts/.s2i/bin/assemble


```
$ cat ~/sunlife-training/developer-src/s2i-scripts/.s2i/bin/assemble
```



```
...output omitted...
######## CUSTOMIZATION STARTS HERE ############

echo "---> Installing application source"
cp -Rf /tmp/src/*.html ./

DATE=`date "+%b %d, %Y @ %H:%M %p"`

echo "---> Creating info page"
echo "Page built on $DATE" >> ./info.html
echo "Proudly served by Apache HTTP Server version $HTTPD_VERSION" >> ./info.html

######## CUSTOMIZATION ENDS HERE ############
...output omitted...
```



### 4. The .s2i/bin/run script changes the default log level of the startup messages in the web server to debug


```
$ cat ~/sunlife-training/developer-src/s2i-scripts/.s2i/bin/run
```



```
# Make Apache show 'debug' level logs during startup
exec run-httpd -e debug $@
```



## Deploy the application to an OpenShift cluster. Verify that the custom S2I scripts are executed


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



### 2. Create a new project &lt;urname>-s2i-scripts


```
$ oc new-project sunny-s2i-scripts
```



```
Now using project "sunny-s2i-scripts" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Create a new application called bonjour from sources in Git. 


```
$ oc new-app --as-deployment-config  --name bonjour  httpd:2.4~https://github.com/<your name>/sunlife-training  --context-dir developer-src/s2i-scripts
```



```
...output omitted...
--> Creating resources ...
imagestream.image.openshift.io "bonjour" created
buildconfig.build.openshift.io "bonjour" created
deploymentconfig.apps.openshift.io "bonjour" created
service "bonjour" created
--> Success
...output omitted...
```



### 4. View the build logs. Wait until the build finishes and the application container image is pushed to the OpenShift registry


```
$ oc logs -f bc/bonjour
```



```
Cloning "https://github.com/youruser/DO288-apps" ...
...output omitted...
---> Enabling s2i support in httpd24 image
AllowOverride All
---> Installing application source
---> Creating info page
Pushing image image-registry.openshift-image-registry.svc:5000/youruser-s2i-scripts/bonjour:latest ... ...
...output omitted...
Push successful
```



## Test the application


### 1. Wait until the application is deployed 


```
$ oc get pod
```



### 2. Expose the application for external access using a route


```
$ oc expose svc bonjour
```



### 3. Obtain the route URL using the oc get route command


```
$  oc get route
```



### 4. Invoke the index page of the application with the curl command 


```
$  curl http://<the route>
```



```
Hello Class! Sunlife Developer rocks!!!
```



### 5. Invoke the index page of the application with the curl command 


```
$  curl http://<the route>/info.html
```



```
Page built on Oct 25, 2020 @ 16:12 PM
Proudly served by Apache HTTP Server version 2.4
```



### 6. Inspect the logs for the application pod


```
$  oc logs bonjour-1-km4bq
```



```
...output omitted...
[Fri Nov 03 16:12:21.690941 2017] [so:debug] [pid 9] mod_so.c(266): AH01575: loaded module systemd_module from /opt/rh/httpd24/root/etc/httpd/modules/mod_systemd.so
[Fri Nov 03 16:12:21.691050 2017] [so:debug] [pid 9] mod_so.c(266): AH01575: loaded module cgi_module from /opt/rh/httpd24/root/etc/httpd/modules/mod_cgi.so
...output omitted...
[Fri Nov 03 16:12:21.742471 2017] [ssl:debug] [pid 9] ssl_engine_init.c(270): AH01886: SSL FIPS mode disabled
...output omitted...
[Fri Nov 03 16:12:21.745520 2017] [mpm_prefork:notice] [pid 9] AH00163: Apache/2.4.25 (Red Hat) OpenSSL/1.0.1e-fips configured -- resuming normal operations
[Fri Nov 03 16:12:21.745530 2017] [core:notice] [pid 9] AH00094: Command line: 'httpd -D FOREGROUND -e debug'
...output omitted...
10.131.0.16 - - [03/Nov/2019:16:28:53 +0000] "GET / HTTP/1.1" 200 28 "-" "curl/7.29.0"
10.128.2.216 - - [03/Nov/2019:16:29:03 +0000] "GET /info.html HTTP/1.1" 200 87 "-" "curl/7.29.0"
```



## Clean up


```
$ oc delete project  <urname>-s2i-scripts
```


