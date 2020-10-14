<!-- Copy and paste the converted output. -->



# RBAC


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



### 2. List all clusterrolebindings that reference the self-provisioner cluster role


```
$ oc get clusterrolebinding -o wide | grep -E 'NAME|self-provisioner'
```



```
NAME                                                                             ROLE                                                                               AGE     USERS                                   GROUPS                                         SERVICEACCOUNTS
self-provisioners                                                                ClusterRole/self-provisioner                                                       2d10h                                           system:authenticated:oauth               
```



### 2. Confirm that the self-provisioners **clusterrolebindings** that you found in the previous step assign the self-provisioner cluster role to the **system:authenticated:oauth** group. 


```
$ oc describe clusterrolebindings self-provisioners
```



```
Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth


```



### 3. Remove the self-provisioner clusterrole from the system:authenticated:oauth virtual group, which deletes the self- provisioners role binding. You can safely ignore the warning about your changes being lost.


```
$ oc adm policy remove-cluster-role-from-group self-provisioner system:authenticated:oauth
```



```
Warning: Your changes may get lost whenever a master is restarted, unless you prevent reconciliation of this rolebinding using the following command: oc annotate clusterrolebinding.rbac self-provisioners 'rbac.authorization.kubernetes.io/autoupdate=false' --overwriteclusterrole.rbac.authorization.k8s.io/self-provisioner removed: "system:authenticated:oauth"
```



### 4. Determine if any other clusterrolebindings reference the self-provisioner cluster role.


```
$ oc get clusterrolebinding -o wide | grep -E 'NAME|self-provisioner'
```



```
NAME                                                                             ROLE                                                                               AGE     USERS                                   GROUPS                                         SERVICEACCOUNTS


```



### 5. Login as the leader user1, using the password from the RHT_OCP4_USER_PASSWD variable, and try to create a project. It should fail.


```
$ oc login -u user1 
```



```
Authentication required for https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443 (openshift)
Username: user1
Password: 
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * user1-bookinfo
    user1-catalog
    user1-cloudnative-pipeline
    user1-cloudnativeapps
    user1-inventory
    user1-istio-system

Using project "user1-bookinfo".
```



```
$ oc new-project test
```



```
Error from server (Forbidden): You may not request a new project via this API.
```



### 6. Login as the admin user and create the **authorization-rbac-&lt;urname>** project.


```
$ oc login -u opentlc-mgr
```



```
Authentication required for https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443 (openshift)
Username: opentlc-mgr
Password: 
Login successful.

You have access to 274 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "user1-bookinfo".
```



```
$ oc new-project authorization-rbac-sunny
```



```
Now using project "authorization-rbac-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```



### 7. Grant project administration privileges to the user1 user on the authorization-rbac-&lt;urname> project.


```
$ oc policy add-role-to-user admin user1
```



```
clusterrole.rbac.authorization.k8s.io/admin added: "user1"
```



### 8. Create a group called dev-group-&lt;urname>


```
$ oc adm groups new dev-group-sunny
```



```
group.user.openshift.io/dev-group-sunny created
```



### 9. Add the user2 user to dev-group-&lt;urname>.


```
$ oc adm groups add-users dev-group-sunny user2
```



```
group.user.openshift.io/dev-group-sunny added: "user2"
```



### 10. Create a second group called qa-group-&lt;urname>


```
$ oc adm groups new qa-group-sunny
```



```
group.user.openshift.io/qa-group-sunny created
```



### 11. Add the user3 user to qa-group-&lt;urname>.


```
$ oc adm groups add-users qa-group-sunny user3
```



```
group.user.openshift.io/qa-group-sunny added: "user3"
```



### 12. Review all existing OpenShift groups to verify that they have the correct members 


```
$ oc get groups
```



```
NAME              USERS
dev-group-sunny   user2
qa-group-sunny    user3
```



### 13. As the user1 user, assign write privileges for dev-group-&lt;urname> and read privileges for qa-group-&lt;urname> to the authorization-rbac-&lt;urname> project.

Login as user1


```
$ oc login -u user1
```



```
Authentication required for https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443 (openshift)
Username: user1
Password: 
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * authorization-rbac-sunny
    user1-bookinfo
    user1-catalog
    user1-cloudnative-pipeline
    user1-cloudnativeapps
    user1-inventory
    user1-istio-system

Using project "authorization-rbac-sunny".
```


Add write privileges to dev-group-&lt;urname> on the authorization-rbac-&lt;urname> project.


```
$ oc policy add-role-to-group edit dev-group-sunny
```



```
clusterrole.rbac.authorization.k8s.io/edit added: "dev-group-sunny"
```


Add read privileges to qa-group-&lt;urname> on the authorization-rbac-&lt;urname> project.


```
$ oc policy add-role-to-group view qa-group-sunny
```



```
clusterrole.rbac.authorization.k8s.io/view added: "qa-group-sunny"
```


Review all rolebindings on the authorization-rbac-&lt;urname> project to verify that they assign roles to the correct groups and users. The following output omits default role bindings assigned by OpenShift to service accounts.


```
$ oc get rolebindings -o wide
```



```
NAME                    ROLE                               AGE     USERS         GROUPS                                            SERVICEACCOUNTS
admin                   ClusterRole/admin                  18m     opentlc-mgr                                                     
admin-0                 ClusterRole/admin                  15m     user1                                                           
edit                    ClusterRole/edit                   18m                                                                     authorization-rbac-sunny/pipeline
edit-0                  ClusterRole/edit                   2m31s                 dev-group-sunny                                   
system:deployers        ClusterRole/system:deployer        18m                                                                     authorization-rbac-sunny/deployer
system:image-builders   ClusterRole/system:image-builder   18m                                                                     authorization-rbac-sunny/builder
system:image-pullers    ClusterRole/system:image-puller    18m                   system:serviceaccounts:authorization-rbac-sunny   
view                    ClusterRole/view                   66s                   qa-group-sunny                                    

```



### 14. As the user2 user, deploy an Apache HTTP Server to prove that it has write privileges in the project. Also try to grant write privileges to the qa-engineer user to prove that the developer user has no project administration privileges.

Login as user2


```
$ oc login -u user2
```



```
Authentication required for https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443 (openshift)
Username: user2
Password: 
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * authorization-rbac-sunny
    user1-bookinfo
    user1-catalog
    user1-cloudnative-pipeline
    user1-cloudnativeapps
    user1-inventory
    user1-istio-system

Using project "authorization-rbac-sunny".
```


Deploy an Apache HTTP Server using the standard imagestream fromOpenShift.


```
$ oc new-app --name httpd httpd:2.4
```



```
warning: Cannot find git. Ensure that it is installed and in your path. Git is required to work with git repositories.
--> Found image 610c566 (3 weeks old) in image stream "openshift/httpd" under tag "2.4" for "httpd:2.4"

    Apache httpd 2.4 
    ---------------- 
    Apache httpd 2.4 available as container, is a powerful, efficient, and extensible web server. Apache supports a variety of features, many implemented as compiled modules which extend the core functionality. These can range from server-side programming language support to authentication schemes. Virtual hosting allows one Apache installation to serve many different Web sites.

    Tags: builder, httpd, httpd24

    * This image will be deployed in deployment config "httpd"
    * Ports 8080/tcp, 8443/tcp will be load balanced by service "httpd"
      * Other containers can access this service through the hostname "httpd"

--> Creating resources ...
    imagestreamtag.image.openshift.io "httpd:2.4" created
    deploymentconfig.apps.openshift.io "httpd" created
    service "httpd" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/httpd' 
    Run 'oc status' to view your app.

```


Try to grant write privileges to the user3 user. It should fail.


```
$ oc policy add-role-to-user edit user3
```



```
Error from server (Forbidden): rolebindings.rbac.authorization.k8s.io is forbidden: User "user2" cannot list resource "rolebindings" in API group "rbac.authorization.k8s.io" in the namespace "authorization-rbac-sunny"
```



### 15. Verify that the user3 user only has read privileges on the httpd application

Login as user3


```
$ oc login -u user3
```



```
Authentication required for https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443 (openshift)
Username: user3
Password: 
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * authorization-rbac-sunny
    user1-bookinfo
    user1-catalog
    user1-cloudnative-pipeline
    user1-cloudnativeapps
    user1-inventory
    user1-istio-system

Using project "authorization-rbac-sunny".
```


Attempt to scale the httpd application. It should fail.


```
$ oc scale dc httpd --replicas 3
```



```
Error from server (Forbidden): deploymentconfigs.apps.openshift.io "httpd" is forbidden: User "user3" cannot patch resource "deploymentconfigs/scale" in API group "apps.openshift.io" in the namespace "authorization-rbac-sunny"
```



### 16. Restore project creation privileges to all users.

Login as the admin user.


```
$ oc login -u opentlc-mgr
```



```
Authentication required for https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443 (openshift)
Username: opentlc-mgr
Password: 
Login successful.


Using project "authorization-rbac-sunny".
```


Restore project creation privileges for all users by recreating the self-provisioners cluster role binding created by the OpenShift installer. You can safely ignore the warning that the group was not found.


```
$ oc adm policy add-cluster-role-to-group --rolebinding-name self-provisioners  self-provisioner system:authenticated:oauth
```



```
Warning: Group 'system:authenticated:oauth' not found
clusterrole.rbac.authorization.k8s.io/self-provisioner added: "system:authenticated:oauth"
```


