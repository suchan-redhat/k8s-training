<!-- Copy and paste the converted output. -->

<!-----
NEW: Check the "Suppress top comment" option to remove this info from the output.

Conversion time: 0.603 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs to Markdown version 1.0β29
* Fri Oct 09 2020 20:09:03 GMT-0700 (PDT)
* Source doc: Lab 1 - Managing Authentication
* Tables are currently converted to HTML tables.
----->



## Create New HTPasswd Authentication



1. Create a new htpasswd file with new user, say sunny

    ```
$ htpasswd -c -B -b  /tmp/htpasswd <user name> <password>
$ htpasswd -c -B  -b  /tmp/htpasswd sunny password
```


2. Review the content of the new file

    ```
$ cat /tmp/htpasswd 
```


3. Login to OpenShift as a cluster administrator 

    ```
$ oc login -u <cluster admin user> <url>
```


4. Update the secret

    ```
$ oc create secret generic htpasswd-secret-<uniq-id> --from-file htpasswd=/tmp/htpasswd -n openshift-config 
$ oc create secret generic htpasswd-secret-sunny --from-file htpasswd=/tmp/htpasswd -n openshift-config 
```


5. Assign admin permission to the user

    ```
$ oc adm policy add-cluster-role-to-user cluster-admin <ur username>
$ oc adm policy add-cluster-role-to-user cluster-admin sunny
```


6. Add the htpasswd into the OAuth config has htpasswd identity provider
    1. Obtain the oauth config from OCP

        ```
$ oc get -o yaml oauth cluster > /tmp/oauth.yaml
```


    2. Edit the oauth.yaml to add new identity provider as below

        ```
- htpasswd:
    fileData:
      name: htpasswd-secret-<uniq-id>
  mappingMethod: claim
  name: sunny_htpasswd
  type: HTPasswd
```


    3. Apply the newly updated resource 

        ```
$ oc replace -f /tmp/oauth.yaml
```


7. Login with newly created user

    ```
$ oc login -u sunny -p password
```


8. Verify the admin right by running administrative commands

    ```
$ oc get nodes
$ oc whoami
```




## User management with HTPasswd Identity Provider


### Create New User



1. Obtain existing HTPasswd file from OCP

    ```
$ oc extract -n openshift-config secret/htpasswd-secret --to - > /tmp/htpasswd-adduser
```


2. Add the new user

    ```
$ htpasswd -b  /tmp/htpasswd-adduser sunny-developer password
```


3. Review the content of the file

    ```
$ cat /tmp/htpasswd-adduser 
```


4. Update the secret

    ```
$ oc create secret generic htpasswd-secret-<uniq-id> --from-file htpasswd=/tmp/htpasswd-adduser  -n openshift-config --dry-run -o yaml | oc replace -f -
$ oc create secret generic htpasswd-secret-sunny --from-file htpasswd=/tmp/htpasswd-adduser -n openshift-config --dry-run -o yaml | oc replace -f -
```


5. Login with newly created user

    ```
$ oc login -u sunny-developer -p password
```


6. Verify the user and make sure he has no admin right

    ```
$ oc whoami
$ oc get node 
```




### Update User Password



1. Obtain existing HTPasswd file from OCP

    ```
$ oc extract -n openshift-config secret/htpasswd-secret --to - > /tmp/htpasswd-update
```


2. Add the new user

    ```
$ htpasswd -b  /tmp/htpasswd-update sunny-developer newpassword
```


3. Review the content of the file

    ```
$ cat /tmp/htpasswd-update 
```


4. Update the secret

    ```
$ oc create secret generic htpasswd-secret-<uniq-id> --from-file htpasswd=/tmp/htpasswd-update  -n openshift-config --dry-run -o yaml | oc replace -f -
$ oc create secret generic htpasswd-secret-sunny --from-file htpasswd=/tmp/htpasswd-update -n openshift-config --dry-run -o yaml | oc replace -f -
```


5. Login with newly created user

    ```
$ oc login -u sunny-developer -p newpassword
```


6. Verify the user and make sure he has no admin right

    ```
$ oc whoami
$ oc get node 
```




### Remove User 



1. Obtain existing HTPasswd file from OCP

    ```
$ oc extract -n openshift-config secret/htpasswd-secret --to - > /tmp/htpasswd-remove
```


2. remove the user

    ```
$ htpasswd -D  /tmp/htpasswd-remove sunny-developer
```


3. Review the content of the file

    ```
$ cat /tmp/htpasswd-update 
```


4. Update the secret

    ```
$ oc create secret generic htpasswd-secret-<uniq-id> --from-file htpasswd=/tmp/htpasswd-remove  -n openshift-config --dry-run -o yaml | oc replace -f -
$ oc create secret generic htpasswd-secret-sunny --from-file htpasswd=/tmp/htpasswd-remove -n openshift-config --dry-run -o yaml | oc replace -f -
```


5. Remove identity

    ```
$ oc delete identity <identityprovidername>:<username>
$ oc delete identity sunny_htpasswd:sunny-developer
```


6. Remove user

    ```
$ oc delete user <username>
$ oc delete user sunny-developer
```


7. Verify the user is removed by login again

    ```
$ oc whoami
$ oc get node 
```




### Remove Identity Provider 



1. Edit oauth

    ```
$ oc edit oauth cluster

<remove the newly added sections>
```


2. remove the secret

    ```
$ oc delete secret generic htpasswd-secret-<uniq-id>  -n openshift-config
$ oc delete secret generic htpasswd-secret-sunny  -n openshift-config
```


3. Verify the identity provider does not exist by login again

    ```
$ oc login -u sunny -p password
```


4. Update the secret

    ```
$ oc create secret generic htpasswd-secret-<uniq-id> --from-file htpasswd=/tmp/htpasswd-remove  -n openshift-config --dry-run -o yaml | oc replace -f -
$ oc create secret generic htpasswd-secret-sunny --from-file htpasswd=/tmp/htpasswd-remove -n openshift-config --dry-run -o yaml | oc replace -f -
```


5. Remove identity

    ```
$ oc delete identity <identityprovidername>:<username>
$ oc delete identity sunny_htpasswd:sunny-developer
```


6. Remove user

    ```
$ oc delete user <username>
$ oc delete user sunny-developer
```


7. Verify the user is removed by login again

    ```
$ oc whoami
$ oc get node 
```


