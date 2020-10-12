<!-- Copy and paste the converted output. -->



# Executing Troubleshooting commands


## Inspect the status of your cluster node


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



### 2. Verify that all nodes on your cluster are ready


```
$ oc get node
```



```
NAME                                              STATUS   ROLES    AGE   VERSION
ip-10-0-141-135.ap-southeast-1.compute.internal   Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-142-187.ap-southeast-1.compute.internal   Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-142-60.ap-southeast-1.compute.internal    Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-145-179.ap-southeast-1.compute.internal   Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-154-68.ap-southeast-1.compute.internal    Ready    master   14h   v1.18.3+012b3ec
ip-10-0-161-101.ap-southeast-1.compute.internal   Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-164-179.ap-southeast-1.compute.internal   Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-166-110.ap-southeast-1.compute.internal   Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-174-165.ap-southeast-1.compute.internal   Ready    master   14h   v1.18.3+012b3ec
ip-10-0-208-2.ap-southeast-1.compute.internal     Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-211-223.ap-southeast-1.compute.internal   Ready    worker   14h   v1.18.3+012b3ec
ip-10-0-216-183.ap-southeast-1.compute.internal   Ready    master   14h   v1.18.3+012b3ec
ip-10-0-223-34.ap-southeast-1.compute.internal    Ready    worker   14h   v1.18.3+012b3ec
```



### 2. Verify whether any of your worker nodes are close to using all of the CPU and memory available to them. 

Use the node-role.kubernetes.io/worker label to filter. Repeat the following command a few times to prove that you see actual usage of CPU and memory from your nodes. The numbers you see should change slightly each time you repeat the command.


```
$ oc adm top node -l node-role.kubernetes.io/worker
```



```
NAME                                              CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
ip-10-0-141-135.ap-southeast-1.compute.internal   941m         6%     6376Mi          10%       
ip-10-0-142-187.ap-southeast-1.compute.internal   908m         5%     6134Mi          9%        
ip-10-0-142-60.ap-southeast-1.compute.internal    651m         4%     6452Mi          10%       
ip-10-0-145-179.ap-southeast-1.compute.internal   1188m        7%     8627Mi          13%       
ip-10-0-161-101.ap-southeast-1.compute.internal   957m         6%     5707Mi          9%        
ip-10-0-164-179.ap-southeast-1.compute.internal   1117m        7%     5758Mi          9%        
ip-10-0-166-110.ap-southeast-1.compute.internal   932m         6%     6154Mi          9%        
ip-10-0-208-2.ap-southeast-1.compute.internal     915m         5%     8413Mi          13%       
ip-10-0-211-223.ap-southeast-1.compute.internal   517m         3%     5392Mi          8%        
ip-10-0-223-34.ap-southeast-1.compute.internal    733m         4%     6886Mi          11%

```



### 3. Pickup one of your worker nodes and pass its name to the oc describe command to verify that all of the conditions that might indicate problems are false.


```
$ oc describe node ip-10-0-141-135.ap-southeast-1.compute.internal
```



```
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Mon, 12 Oct 2020 11:49:29 +0800   Sun, 11 Oct 2020 21:32:28 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Mon, 12 Oct 2020 11:49:29 +0800   Sun, 11 Oct 2020 21:32:28 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Mon, 12 Oct 2020 11:49:29 +0800   Sun, 11 Oct 2020 21:32:28 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Mon, 12 Oct 2020 11:49:29 +0800   Sun, 11 Oct 2020 21:34:38 +0800   KubeletReady                 kubelet is posting ready status
```



## Review the logs of Core Components


### 1. List all pods inside the openshift-image-registry project


```
$ oc get pod -n openshift-image-registry
```



```
NAME                                               READY   STATUS      RESTARTS   AGE
cluster-image-registry-operator-5f47f6fcf7-sc2j4   2/2     Running     0          14h
image-pruner-1602460800-n2n4r                      0/1     Completed   0          4h1m
…
…
…
...
```



### 2. Follow the logs of the operator pod (cluster-image-registry-operator-xxx).

The following command fails because that pod has two containers.


```
$ oc logs --tail 3 -n openshift-image-registry cluster-image-registry-operator-5f47f6fcf7-sc2j4
```



```
Error from server (BadRequest): a container name must be specified for pod cluster-image-registry-operator-5f47f6fcf7-sc2j4, choose one of: [cluster-image-registry-operator cluster-image-registry-operator-watch]
```



### 3. Follow the logs of the first container of the operator pod.


```
$ oc logs --tail 3 -n openshift-image-registry -c cluster-image-registry-operator cluster-image-registry-operator-5f47f6fcf7-sc2j4
```



```
I1012 04:03:53.916795      15 controllerimagepruner.go:309] event from image pruner workqueue successfully processed
I1012 04:03:53.973240      15 controller.go:314] event from workqueue successfully processed
I1012 04:03:54.574175      15 controller.go:314] event from workqueue successfully processed
```



### 4. Follow the logs of the image registry server pod (image-registry-xxx from the output of the oc get pod command run previously)


```
$ oc logs --tail 3 -n openshift-image-registry image-registry-55f945484-l2wh8
```



```
time="2020-10-12T04:06:36.232138479Z" level=info msg=response go.version=go1.13.4 http.request.host="10.131.0.6:5000" http.request.id=fee84671-3299-4b31-bbac-516796c901d8 http.request.method=GET http.request.remoteaddr="10.131.0.1:51232" http.request.uri=/healthz http.request.useragent=kube-probe/1.18+ http.response.duration="39.848µs" http.response.status=200 http.response.written=0
time="2020-10-12T04:06:37.801537183Z" level=info msg=response go.version=go1.13.4 http.request.host="10.131.0.6:5000" http.request.id=f58f6863-190b-400f-b056-22547657b70a http.request.method=GET http.request.remoteaddr="10.131.0.1:51256" http.request.uri=/healthz http.request.useragent=kube-probe/1.18+ http.response.duration="59.812µs" http.response.status=200 http.response.written=0
time="2020-10-12T04:06:46.232120995Z" level=info msg=response go.version=go1.13.4 http.request.host="10.131.0.6:5000" http.request.id=0e188eb3-e9b7-4480-97db-1bddcab9aa40 http.request.method=GET http.request.remoteaddr="10.131.0.1:51390" http.request.uri=/healthz http.request.useragent=kube-probe/1.18+ http.response.duration="54.346µs" http.response.status=200 http.response.written=0
```



### 5. Follow the logs of the Kubelet from the same node that you inspected for CPU and memory usage in the previous step


```
$ oc adm node-logs --tail 3 -u kubelet ip-10-0-141-135.ap-southeast-1.compute.internal
```



```
Data from the specified boot (-1) is not available: No such boot ID in journal
-- Logs begin at Sun 2020-10-11 18:13:15 UTC, end at Mon 2020-10-12 04:08:26 UTC. --
Oct 12 04:08:26.040417 ip-10-0-141-135 hyperkube[1613]: I1012 04:08:26.040393    1613 log_handler.go:34] AWS API Send: ec2metadata GetMetadata &{GetMetadata GET /meta-data/public-hostname <nil> <nil>} <nil>
Oct 12 04:08:26.040417 ip-10-0-141-135 hyperkube[1613]: I1012 04:08:26.040414    1613 log_handler.go:39] AWS API ValidateResponse: ec2metadata GetMetadata &{GetMetadata GET /meta-data/public-hostname <nil> <nil>} <nil> 404 Not Found
Oct 12 04:08:26.040529 ip-10-0-141-135 hyperkube[1613]: I1012 04:08:26.040467    1613 aws.go:1473] Could not determine public DNS from AWS metadata.
http.request.remoteaddr="10.131.0.1:51390" http.request.uri=/healthz http.request.useragent=kube-probe/1.18+ http.response.duration="54.346µs" http.response.status=200 http.response.written=0
```



## Start a shell session to node


### 1. Start a shell session on the working node, and then use the chroot command to enter the local file system of the host.


```
$ oc debug node/ip-10-0-141-135.ap-southeast-1.compute.internal
```



```
Starting pod/ip-10-0-141-135ap-southeast-1computeinternal-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.141.135
If you don't see a command prompt, try pressing enter.
sh-4.2# chroot /host
sh-4.2# 
```



### 2. Still using the same shell session, verify that the Kubelet and the CRI-O container engine are running. Type q to exit the command.


```
sh-4.4# systemctl status kubelet
● kubelet.service - MCO environment configuration
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-mco-default-env.conf
   Active: active (running) since Sun 2020-10-11 13:32:15 UTC; 14h ago
  Process: 1611 ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state (code=exited, status=0/SUCCESS)
  Process: 1609 ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
 Main PID: 1613 (kubelet)
    Tasks: 45 (limit: 406641)
   Memory: 340.3M
      CPU: 2h 13min 48.991s
   CGroup: /system.slice/kubelet.service
           └─1613 kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/var/lib/kubelet/kubeconfig --container-runtime=remote --c>

Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.648604    1613 exec.go:60] Exec probe response: "OK"
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.648646    1613 prober.go:133] Readiness probe for "istio-galley-6db5465bf6-fdxkm_user4-istio-system(7a22b0ca-6545-4965-a3>
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.741661    1613 prober.go:181] HTTP-Probe Host: http://10.128.6.47, Port: 15020, Path: /healthz/ready
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.741685    1613 prober.go:184] HTTP-Probe Headers: map[]
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.743203    1613 http.go:128] Probe succeeded for http://10.128.6.47:15020/healthz/ready, Response: {200 OK 200 HTTP/1.1 1 >
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.743240    1613 prober.go:133] Readiness probe for "istio-egressgateway-5b8f5d55db-6pdbh_user23-istio-system(0f32a437-9dcf>
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.824362    1613 prober.go:166] Exec-Probe Pod: istio-galley-5587fc9cd-75kvz, Container: galley, Command: [/usr/local/bin/g>
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.910361    1613 exec.go:60] Exec probe response: "OK"
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.910387    1613 prober.go:133] Readiness probe for "istio-galley-5587fc9cd-75kvz_user29-istio-system(78851b21-5628-4150-9f>
Oct 12 04:13:22 ip-10-0-141-135 hyperkube[1613]: I1012 04:13:22.910367    1613 reflector.go:495] object-"user4-istio-system"/"istio-galley-service-account-dockercfg-7528v": Watch close >
```


Rerun the same command against the cri-o service. Type q to exit from the command.


```
sh-4.4# systemctl status cri-o  
● crio.service - MCO environment configuration
   Loaded: loaded (/usr/lib/systemd/system/crio.service; disabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/crio.service.d
           └─10-mco-default-env.conf
   Active: active (running) since Sun 2020-10-11 13:32:15 UTC; 14h ago
     Docs: https://github.com/cri-o/cri-o
 Main PID: 1566 (crio)
    Tasks: 41
   Memory: 11.1G
      CPU: 2h 55min 7.522s
   CGroup: /system.slice/crio.service
           └─1566 /usr/bin/crio --enable-metrics=true --metrics-port=9537

Oct 12 04:10:32 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:32.226414375Z" level=info msg="Pulling image: registry.redhat.io/rhel7/support-tools:latest" id=9fd4db69-a648-421f-9a1>
Oct 12 04:10:42 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:42.877396612Z" level=info msg="Pulled image: registry.redhat.io/rhel7/support-tools@sha256:6cca005ebb5a083f9af7ec3622d>
Oct 12 04:10:42 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:42.878346566Z" level=info msg="Checking image status: registry.redhat.io/rhel7/support-tools" id=5e13b32c-3cb5-4579-bd>
Oct 12 04:10:42 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:42.879125696Z" level=info msg="Image status: &ImageStatusResponse{Image:&Image{Id:5e8cf4b604a2644343b11b377f74084daad1>
Oct 12 04:10:42 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:42.879822693Z" level=info msg="Creating container: default/ip-10-0-141-135ap-southeast-1computeinternal-debug/containe>
Oct 12 04:10:43 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:43.040803003Z" level=info msg="Created container ea4e460928badc25d21143c937002e4b333eb799cb201971b38b7e8baa4b2c81: def>
Oct 12 04:10:43 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:43.041223270Z" level=info msg="Starting container: ea4e460928badc25d21143c937002e4b333eb799cb201971b38b7e8baa4b2c81" i>
Oct 12 04:10:43 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:10:43.053733960Z" level=info msg="Started container ea4e460928badc25d21143c937002e4b333eb799cb201971b38b7e8baa4b2c81: def>
Oct 12 04:12:16 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:12:16.519442220Z" level=info msg="Checking image status: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:eb9ab6f214>
Oct 12 04:12:16 ip-10-0-141-135 crio[1566]: time="2020-10-12 04:12:16.520469941Z" level=info msg="Image status: &ImageStatusResponse{Image:&Image{Id:e66662827187986d2c58eba25a6300d4c792>

```



### 3. Still using the same shell session,verify that the openvswitch pod is running.


```
sh-4.4# crictl ps --name openvswitch
CONTAINER           IMAGE                                                                                                                    CREATED             STATE               NAME                ATTEMPT             POD ID
30c087ca88918       quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:e8b0ea7dac1433c9f7c31b27fe548cc4b8dd731ead0e5a8b12fca08ba79ccad0   15 hours ago        Running             openvswitch         0                   ce581587d1078
```


 \
4. Terminate the chroot session and shell session to the node. This also terminates the oc debug node command.


```
sh-4.4# exit
exit
sh-4.2# exit
exit

Removing debug pod ...
```



## Troubleshooting basic deployment issue


### 1. Create a new project with problem deployments


```
$ oc new-project exec-troubleshoot-<yourname>
$ oc new-project exec-troubleshoot-sunny
```



```
Now using project "exec-troubleshoot-sunny" on server "https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```


Create an app with wrong image for demo purpose


```
$ oc new-app  registry.access.redhat.com/rhscl/postgresq-96-rhel7:1 -e POSTGRESQL_ADMIN_PASSWORD=password --allow-missing-images$ 
```



### 2. Verify that the project has a single pod in either the ErrImagePull or ImagePullBackOff status.


```
$ oc get pod
```



```
NAME                          READY   STATUS         RESTARTS   AGE
postgresq-96-rhel7-1-deploy   1/1     Running        0          111s
postgresq-96-rhel7-1-zwxn2    0/1     ErrImagePull   0          99s
```



### 3. Verify that the project includes a Kubernetes deployment that manages the pod.


```
$ oc status
```



```
In project exec-troubleshoot-sunny on server https://api.cluster-sunlife-2bfb.sunlife-2bfb.sandbox1899.opentlc.com:6443

dc/postgresq-96-rhel7 deploys registry.access.redhat.com/rhscl/postgresq-96-rhel7:1 
  deployment #1 running for 2 minutes - 0/1 pods


2 infos identified, use 'oc status --suggest' to see details.
```



### 4. List all events from the current project  and look for error messages related to the pod.


```
$ oc get events
```



```
LAST SEEN   TYPE      REASON              OBJECT                                       MESSAGE
4m37s       Normal    Scheduled           pod/postgresq-96-rhel7-1-deploy              Successfully assigned exec-troubleshoot-sunny/postgresq-96-rhel7-1-deploy to ip-10-0-223-34.ap-southeast-1.compute.internal
4m35s       Normal    AddedInterface      pod/postgresq-96-rhel7-1-deploy              Add eth0 [10.131.0.57/23]
4m35s       Normal    Pulling             pod/postgresq-96-rhel7-1-deploy              Pulling image "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:17a0c1d7e410c74d90a1051b495c3a380f3c1c2ff3ea8a69355687a1ebb16921"
4m25s       Normal    Pulled              pod/postgresq-96-rhel7-1-deploy              Successfully pulled image "quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:17a0c1d7e410c74d90a1051b495c3a380f3c1c2ff3ea8a69355687a1ebb16921"
4m25s       Normal    Created             pod/postgresq-96-rhel7-1-deploy              Created container deployment
4m25s       Normal    Started             pod/postgresq-96-rhel7-1-deploy              Started container deployment
4m25s       Normal    Scheduled           pod/postgresq-96-rhel7-1-zwxn2               Successfully assigned exec-troubleshoot-sunny/postgresq-96-rhel7-1-zwxn2 to ip-10-0-223-34.ap-southeast-1.compute.internal
4m23s       Normal    AddedInterface      pod/postgresq-96-rhel7-1-zwxn2               Add eth0 [10.131.0.58/23]
3m2s        Normal    Pulling             pod/postgresq-96-rhel7-1-zwxn2               Pulling image "registry.access.redhat.com/rhscl/postgresq-96-rhel7:1"
3m2s        Warning   Failed              pod/postgresq-96-rhel7-1-zwxn2               Failed to pull image "registry.access.redhat.com/rhscl/postgresq-96-rhel7:1": rpc error: code = Unknown desc = Error reading manifest 1 in registry.access.redhat.com/rhscl/postgresq-96-rhel7: name unknown: Repo not found
3m2s        Warning   Failed              pod/postgresq-96-rhel7-1-zwxn2               Error: ErrImagePull
2m34s       Normal    BackOff             pod/postgresq-96-rhel7-1-zwxn2               Back-off pulling image "registry.access.redhat.com/rhscl/postgresq-96-rhel7:1"
2m49s       Warning   Failed              pod/postgresq-96-rhel7-1-zwxn2               Error: ImagePullBackOff
4m25s       Normal    SuccessfulCreate    replicationcontroller/postgresq-96-rhel7-1   Created pod: postgresq-96-rhel7-1-zwxn2
4m37s       Normal    DeploymentCreated   deploymentconfig/postgresq-96-rhel7          Created new replication controller "postgresq-96-rhel7-1" for version 1
```



### 5. Use Skopeo to find information about the container image from the events.


```
$ skopeo inspect docker://registry.access.redhat.com/rhscl/postgresq-96-rhel7:1
```



```
FATA[0001] Error parsing image name "docker://registry.access.redhat.com/rhscl/postgresq-96-rhel7:1": Error reading manifest 1 in registry.access.redhat.com/rhscl/postgresq-96-rhel7: name unknown: Repo not found 
```



### 6. Verify that it works if you replace postgresq-96-rhel7 with postgresql-96-rhel7


```
$ skopeo inspect docker://registry.access.redhat.com/rhscl/postgresql-96-rhel7:1
```



```
{
    "Name": "registry.access.redhat.com/rhscl/postgresql-96-rhel7",
    "Digest": "sha256:6c3090779c0553d773380ff4cb0532086075376964de398417d494313f9a933b",
    "RepoTags": [
        "1-42.1561731092",
        "1-59",
        "1-58",
        "1-17",
        "1-10",
        "1-13",
        "1-12",
        "1-51",
        "1-53",
        "1-52",
        "1-57",
        "1-38",
        "1-37",
        "1-36",
        "1-32",
        "1",
        "1-25.1535384678",
        "1-8",
        "1-5",
        "1-4",
        "1-7",
        "1-6",
        "1-61",
        "1-63",
        "1-48",
        "1-49",
        "1-46",
        "1-47",
        "1-44",
        "1-45",
        "1-42",
        "1-40",
        "1-51.1575996559",
        "1-28",
        "1-14",
        "1-25",
        "1-27",
        "latest"
    ],
    "Created": "2020-09-21T15:39:59.766636Z",
    "DockerVersion": "1.13.1",
    "Labels": {
        "architecture": "x86_64",
        "build-date": "2020-09-21T15:38:28.666467",
        "com.redhat.build-host": "cpt-1002.osbs.prod.upshift.rdu2.redhat.com",
        "com.redhat.component": "rh-postgresql96-container",
        "com.redhat.license_terms": "https://www.redhat.com/en/about/red-hat-end-user-license-agreements#rhel",
        "description": "PostgreSQL is an advanced Object-Relational database management system (DBMS). The image contains the client and server programs that you'll need to create, run, maintain and access a PostgreSQL DBMS server.",
        "distribution-scope": "public",
        "io.k8s.description": "PostgreSQL is an advanced Object-Relational database management system (DBMS). The image contains the client and server programs that you'll need to create, run, maintain and access a PostgreSQL DBMS server.",
        "io.k8s.display-name": "PostgreSQL 9.6",
        "io.openshift.expose-services": "5432:postgresql",
        "io.openshift.s2i.assemble-user": "26",
        "io.openshift.s2i.scripts-url": "image:///usr/libexec/s2i",
        "io.openshift.tags": "database,postgresql,postgresql96,rh-postgresql96",
        "io.s2i.scripts-url": "image:///usr/libexec/s2i",
        "maintainer": "SoftwareCollections.org \u003csclorg@redhat.com\u003e",
        "name": "rhscl/postgresql-96-rhel7",
        "release": "63",
        "summary": "PostgreSQL is an advanced Object-Relational database management system",
        "url": "https://access.redhat.com/containers/#/registry.access.redhat.com/rhscl/postgresql-96-rhel7/images/1-63",
        "usage": "podman run -d --name postgresql_database -e POSTGRESQL_USER=user -e POSTGRESQL_PASSWORD=pass -e POSTGRESQL_DATABASE=db -p 5432:5432 rhscl/postgresql-96-rhel7",
        "vcs-ref": "ae9489bc8bf8f1c543d15dbe9ed5b9d1b6adcacd",
        "vcs-type": "git",
        "vendor": "Red Hat, Inc.",
        "version": "1"
    },
    "Architecture": "amd64",
    "Os": "linux",
    "Layers": [
        "sha256:1323a241cc068f2816dd88f00168be73339471d6dc6eb2e6c761b63b734501b6",
        "sha256:2bd25ca124579d6fce8668ff5d4ed83866d7e7438cb561a51ddde8cc40272822",
        "sha256:5d011ac93e7456d4c646b0fbb53712598bda9a6b0d027b2be788016e078ded77",
        "sha256:16634824fe3f2f2bbbf87f9a3082ec4b3d1b3b084ce14b3e63419a4dc5b3150d"
    ],
    "Env": [
        "PATH=/opt/app-root/src/bin:/opt/app-root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "container=oci",
        "SUMMARY=PostgreSQL is an advanced Object-Relational database management system",
        "DESCRIPTION=PostgreSQL is an advanced Object-Relational database management system (DBMS). The image contains the client and server programs that you'll need to create, run, maintain and access a PostgreSQL DBMS server.",
        "STI_SCRIPTS_URL=image:///usr/libexec/s2i",
        "STI_SCRIPTS_PATH=/usr/libexec/s2i",
        "APP_ROOT=/opt/app-root",
        "HOME=/var/lib/pgsql",
        "PLATFORM=el7",
        "BASH_ENV=/usr/share/container-scripts/postgresql/scl_enable",
        "ENV=/usr/share/container-scripts/postgresql/scl_enable",
        "PROMPT_COMMAND=. /usr/share/container-scripts/postgresql/scl_enable",
        "POSTGRESQL_VERSION=9.6",
        "POSTGRESQL_PREV_VERSION=9.5",
        "PGUSER=postgres",
        "APP_DATA=/opt/app-root",
        "CONTAINER_SCRIPTS_PATH=/usr/share/container-scripts/postgresql",
        "ENABLED_COLLECTIONS=rh-postgresql96"
    ]
} 
```



### 7. Correct the name of the container image


```
$ oc edit dc/postgresq-96-rhel7
```



```
...output omitted...
    spec:
      containers:
      - image: registry.access.redhat.com/rhscl/postgresql-96-rhel7:1
        imagePullPolicy: IfNotPresent
        name: postgresq-96-rhel7
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
...output omitted...
```



### 7. Verify all pods running


```
$ oc get pods
```



```
[root@testocp42 ~]# oc get pods
NAME                          READY   STATUS             RESTARTS   AGE
postgresq-96-rhel7-2-deploy   1/1     Running            0          77s
postgresq-96-rhel7-2-ktj95    1/1     Running   0          74s

```


