<!-- Copy and paste the converted output. -->



# Deploying a Containerized Nexus Server


## Review the application source code


### 1. Preparing your Repo


```
$ git clone https://github.com/<ur own repo>/sunlife-training.git
```



### 2. Inspect the Dockerfile file inside ~/sunlife-training/developer-src/nexus3


```
$ cat ~/sunlife-training/developer-src/nexus3/Dockerfile
```



```
# Copyright (c) 2016-present Sonatype, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM registry.access.redhat.com/ubi8/ubi:8.0

LABEL name="Nexus Repository Manager" \
      vendor=Sonatype \
      version="3.18.0-01" \
      release="3.18.0" \
      url="https://sonatype.com" \
      summary="The Nexus Repository Manager server \
          with universal support for popular component formats." \
      description="The Nexus Repository Manager server \
          with universal support for popular component formats." \
      run="docker run -d --name NAME \
          -p 8081:8081 \
          IMAGE" \
      stop="docker stop NAME" \
      com.sonatype.license="Apache License, Version 2.0" \
      com.sonatype.name="Nexus Repository Manager base image" \
      io.k8s.description="The Nexus Repository Manager server \
          with universal support for popular component formats." \
      io.k8s.display-name="Nexus Repository Manager" \
      io.openshift.expose-services="8081:8081" \
      io.openshift.tags="Sonatype,Nexus,Repository Manager"


ARG NEXUS_VERSION=3.18.0-01
ARG NEXUS_DOWNLOAD_URL=https://download.sonatype.com/nexus/3/nexus-${NEXUS_VERSION}-unix.tar.gz
ARG NEXUS_DOWNLOAD_SHA256_HASH=e1d9d84d8b169b2f6c735e7db35e3310cf9e242da12b4af83da4e3618acfc99e

# configure nexus runtime
ENV SONATYPE_DIR=/opt/sonatype
ENV NEXUS_HOME=${SONATYPE_DIR}/nexus \
    NEXUS_DATA=/nexus-data \
    NEXUS_CONTEXT='' \
    SONATYPE_WORK=${SONATYPE_DIR}/sonatype-work \
    DOCKER_TYPE='rh-docker'

ARG NEXUS_REPOSITORY_MANAGER_COOKBOOK_VERSION="release-0.5.20190212-155606.d1afdfe"
ARG NEXUS_REPOSITORY_MANAGER_COOKBOOK_URL="https://github.com/sonatype/chef-nexus-repository-manager/releases/download/${NEXUS_REPOSITORY_MANAGER_COOKBOOK_VERSION}/chef-nexus-repository-manager.tar.gz"

ADD solo.json.erb /var/chef/solo.json.erb

# Install using chef-solo
# Chef version locked to avoid needing to accept the EULA on behalf of whomever builds the image
RUN yum install -y --disableplugin=subscription-manager hostname procps \
    && curl -L https://www.getchef.com/chef/install.sh | bash -s -- -v 14.12.9 \
    && /opt/chef/embedded/bin/erb /var/chef/solo.json.erb > /var/chef/solo.json \
    && chef-solo \
       --recipe-url ${NEXUS_REPOSITORY_MANAGER_COOKBOOK_URL} \
       --json-attributes /var/chef/solo.json \
    && rpm -qa *chef* | xargs rpm -e \
    && rm -rf /etc/chef \
    && rm -rf /opt/chefdk \
    && rm -rf /var/cache/yum \
    && rm -rf /var/chef \
    && yum clean all

VOLUME ${NEXUS_DATA}

EXPOSE 8081
USER nexus

ENV INSTALL4J_ADD_VM_PARAMS="-Xms1200m -Xmx1200m -XX:MaxDirectMemorySize=2g -Djava.util.prefs.userRoot=${NEXUS_DATA}/javaprefs"

ENTRYPOINT ["/uid_entrypoint.sh"]
CMD ["sh", "-c", "${SONATYPE_DIR}/start-nexus-repository-manager.sh"]
```



### 3. Build the container image and push it to your personal account at Quay.io


```
$ sudo podman build -t nexus3 .
```



```
...output omitted...
STEP 33: COMMIT nexus3
```



```
$ sudo podman login -u <ur name> quay.io
```



```
Password:
Login Succeeded!
```



```
$ sudo skopeo copy  containers-storage:localhost/nexus3  docker://quay.io/<ur name>/nexus3
```



```
...output omitted...
Writing manifest to image destination
Storing signatures
```



## Complete the template to deploy Nexus as a service


### 1. Preparing below template file


```
apiVersion: v1
kind: Template
metadata:
  name: nexus3
objects:
- apiVersion: v1
  kind: ImageStream
  metadata:
    name: ${NEXUS_SERVICE_NAME}
  spec:
    lookupPolicy:
      local: true
    tags:
    - annotations: null
      from:
        kind: DockerImage
        name: quay.io/suchan/nexus3:latest
      name: latest
- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    annotations:
    generation: 1
    labels:
      app: ${NEXUS_SERVICE_NAME}
    name: ${NEXUS_SERVICE_NAME}
  spec:
    replicas: 1
    selector:
      app: ${NEXUS_SERVICE_NAME}
      deploymentconfig: ${NEXUS_SERVICE_NAME}
    strategy:
      activeDeadlineSeconds: 21600
      resources: {}
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        annotations:
        labels:
          app: ${NEXUS_SERVICE_NAME}
          deploymentconfig: ${NEXUS_SERVICE_NAME}
      spec:
        containers:
        - env:
            - name: INSTALL4J_ADD_VM_PARAMS
              value: REPLACE_JVM_OPTS
          imagePullPolicy: Always
          livenessProbe:
            exec:
              command:
              - /bin/sh
              - "-c"
              - >
               curl -siu admin:$(cat /nexus-data/admin.password)
               http://localhost:8081/service/metrics/healthcheck |
               grep healthy | grep true
            failureThreshold: 3
            initialDelaySeconds: REPLACE_DELAY
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: REPLACE_TIMEOUT
          name: ${NEXUS_SERVICE_NAME}
          ports:
          - containerPort: 8081
            protocol: TCP
          readinessProbe:
            exec:
              command:
              - /bin/sh
              - "-c"
              - >
               curl -siu admin:$(cat /nexus-data/admin.password)
               http://localhost:8081/service/metrics/ping |
               grep pong
            failureThreshold: 3
            initialDelaySeconds: REPLACE_DELAY
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: REPLACE_TIMEOUT
          resources:
            limits:
              cpu: "1"
              memory: 2Gi
            requests:
              cpu: 500m
              memory: 256Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
          - mountPath: REPLACE_VOLUME_MOUNT
            name: REPLACE_VOLUME_NAME
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
        volumes:
        - name: REPLACE_VOLUME_NAME
          persistentVolumeClaim:
            claimName: REPLACE_CLAIM_NAME
    test: false
    triggers:
    - type: ConfigChange
    - imageChangeParams:
        automatic: true
        containerNames:
        - ${NEXUS_SERVICE_NAME}
        from:
          kind: ImageStreamTag
          name: nexus3:latest
      type: ImageChange
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
    labels:
      app: ${NEXUS_SERVICE_NAME}
    name: ${NEXUS_SERVICE_NAME}
  spec:
    ports:
    - name: 8081-tcp
      port: 8081
      protocol: TCP
      targetPort: 8081
    selector:
      app: ${NEXUS_SERVICE_NAME}
      deploymentconfig: ${NEXUS_SERVICE_NAME}
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
- apiVersion: v1
  kind: Route
  metadata:
    labels:
      app: ${NEXUS_SERVICE_NAME}
    name: ${NEXUS_SERVICE_NAME}
  spec:
    host: ${HOSTNAME}
    port:
      targetPort: 8081-tcp
    to:
      kind: Service
      name: ${NEXUS_SERVICE_NAME}
      weight: 100
    wildcardPolicy: None
  status:
    ingress:
    - conditions:
      - lastTransitionTime: 2017-11-29T16:57:34Z
        status: "True"
        type: Admitted
      host: ${HOSTNAME}
      routerName: router
      wildcardPolicy: None
- apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    labels:
      app: ${NEXUS_SERVICE_NAME}
    name: REPLACE_PVC
  spec:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
  status: {}
parameters:
- name: HOSTNAME
  displayName: Aplication Hostname
  description: FQDN of the route to access the application
  required: true

- name: NEXUS_SERVICE_NAME
  displayName: Nexus Service Name
  description: The name of the OpenShift Service exposed for the nexus server.
  required: true
  value: nexus3
```



### 2. Inspect the nexus-template.yaml file to verify that it uses the container image built previously


```
$ grep -A1 "kind: DockerImage" ~/nexus-template.yaml 
```



```
        kind: DockerImage
        name: quay.io/youruser/nexus3:latest
```



### 3. Inspect the nexus-template.yaml file to verify that it defines resource request and resource limits for the application pod


```
$ grep -B1 -A5 limits: ~/nexus-template.yaml 
```



```
          resources:
            limits:
              cpu: "1"
              memory: 2Gi
            requests:
              cpu: 500m
              memory: 256Mi
```



### 4. Open the nexus-template.yaml file with a text editor, and then override the INSTALL4J_ADD_VM_PARAMS environment variable to not include any JVM heap sizing configuration, and to define only the Java system properties required by the application.


```
...output omitted...
- apiVersion: v1
  kind: DeploymentConfig
  ...output omitted...
        - env:
            - name: INSTALL4J_ADD_VM_PARAMS
              value: -Djava.util.prefs.userRoot=/nexus-data/javaprefs
...output omitted...
```



### 5. Review the liveness probe in the deployment configuration resource


```
...output omitted...
- apiVersion: v1
  kind: DeploymentConfig
  ...output omitted...
          livenessProbe:
            exec:
              command:
              - /bin/sh
              - "-c"
              - >
               curl -siu admin:$(cat /nexus-data/admin.password)
               http://localhost:8081/service/metrics/healthcheck |
               grep healthy | grep true
            ...output omitted...
            initialDelaySeconds: 120
            ...output omitted...
            timeoutSeconds: 30
...output omitted...
```



### 6. Review the readiness probe in the deployment configuration resource


```
...output omitted...
- apiVersion: v1
  kind: DeploymentConfig
  ...output omitted...
          readinessProbe:
            exec:
              command:
              - /bin/sh
              - "-c"
              - >
               curl -siu admin:$(cat /nexus-data/admin.password)
               http://localhost:8081/service/metrics/ping |
               grep pong
            ...output omitted...
            initialDelaySeconds: 120
            ...output omitted...
            timeoutSeconds: 30
...output omitted...
```



### 7. In the deployment configuration resource, configure the volume mount to use the nexus-data volume at a mount path of /nexus-data


```
...output omitted...
- apiVersion: v1
  kind: DeploymentConfig
  ...output omitted...
          volumeMounts:
          - mountPath: /nexus-data
            name: nexus-data
...output omitted...

```



### 8. In the deployment configuration resource, name the volume nexus-data, and then configure the volume to use the nexus-data-pvc persistent volume claim


```
...output omitted...
- apiVersion: v1
  kind: DeploymentConfig
  ...output omitted...
        volumes:
        - name: nexus-data
          persistentVolumeClaim:
            claimName: nexus-data-pvc
...output omitted...
```



### 9. A persistent volume claim is required to persist Nexus data. In the persistent volume claim resource, name the persistent volume claim nexus-data-pvc by updating the metadata.name attribute


```
...output omitted...
- apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    labels:
      app: ${NEXUS_SERVICE_NAME}
    name: nexus-data-pvc
...output omitted...
```



## Create a new project. Add a secret that allows any project user to pull the Nexus container image from Quay.io 


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



### 2. Create a new project &lt;urname>-nexus-service


```
$ oc new-project sunny-nexus-service
```



```
Now using project "sunny-nexus-service" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

```



### 3. Use Podman without the sudo command to log in to Quay.io.


```
$ podman login -u <urname> quay.io
```



```
Password:
Login Succeeded!
```



### 4. Create a secret to access your Quay.io personal account


```
$ c create secret generic quayio  --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json  --type kubernetes.io/dockerconfigjson
```



```
secret/quayio created
```



### 5. Link the secret to the default service account


```
$ oc secrets link default quayio --for pull
```



## Create a new application


### 1. Create a new application called nexus3 from the template file


```
$ oc new-app --as-deployment-config --name nexus3  -f ~/nexus-template.yaml  -p HOSTNAME=nexus-${RHT_OCP4_DEV_USER}.${RHT_OCP4_WILDCARD_DOMAIN}

```



```
...output omitted...
--> Creating resources ...
imagestream.image.openshift.io "nexus3" created
deploymentconfig.apps.openshift.io "nexus3" created
service "nexus3" created
route.route.openshift.io "nexus3" created
persistentvolumeclaim "nexus-data-pvc" created
--> Success
...output omitted...
```



### 2. Wait until the application pod is running, but not ready


```
$ oc get pods
$ oc logs -f nexus3-1-kfwwh
```



### 3. Wait for the application to be ready and running


```
$ oc get pods
```



## Test the Nexus application.


### 1. Retrieve the route for the Nexus application


```
$ oc get route
```



### 2. Open a web browser and navigate to application route


## Clean up


```
$ oc delete project  <urname>-app-deploy
```


