<!-- Copy and paste the converted output. -->



## Executing Troubleshooting Commands


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



### 2. Pickup one of your worker nodes and pass its name to the oc describe command to verify that all of the conditions that might indicate problems are false.


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


