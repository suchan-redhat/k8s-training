### install helm
```bash
curl https://helm.baltorepo.com/organization/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
 helm repo add stable https://kubernetes-charts.storage.googleapis.com/

```bash
### Install metrics serve

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

### install prometheus
```bash
helm install prometheus  stable/prometheus-operator --namespace monitoring
```

### get grafana password
```bash
kubectl -n monitoring get secret prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```


### create ingress
```bash
cat <<EOF | kubectl -n monitoring apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: grafana
spec:
  rules:
  - host: grafana.192.168.87.245.nip.io
    http:
      paths:
      - backend:
          serviceName: prometheus-grafana
          servicePort: 80
EOF
```

### install nfs server on master
```bash
sudo apt install nfs-kernel-server
sudo mkdir -p /mnt/nfs_share
sudo chown -R nobody:nogroup /mnt/nfs_share/
sudo bash -c 'echo "/mnt/nfs_share  *(rw,sync,no_subtree_check)" >> /etc/exports'
mkdir /mnt/nfs_share/es_data
mkdir /mnt/nfs_share/es_master
chmod 0777 mkdir /mnt/nfs_share/es_datbash
sudo apt install nfs-kernel-server
sudo mkdir -p /mnt/nfs_share
sudo chown -R nobody:nogroup /mnt/nfs_share/
sudo bash -c 'echo "/mnt/nfs_share  *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports'
mkdir /mnt/nfs_share/es_data
mkdir /mnt/nfs_share/es_master
chmod 0777 /mnt/nfs_share/es_data
chmod 0777 /mnt/nfs_share/es_master
mkdir /mnt/nfs_share/es_data_1
mkdir /mnt/nfs_share/es_master_1
chmod 0777 /mnt/nfs_share/es_data_1
chmod 0777 /mnt/nfs_share/es_master_1
mkdir /mnt/nfs_share/es_master_2
chmod 0777 /mnt/nfs_share/es_master_2

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-data
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: logging
    name: data-elasticsearch-data-0
  nfs:
    path: /mnt/nfs_share/es_data
    server: 192.168.87.245
    readOnly: false
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-data-1
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: logging
    name: data-elasticsearch-data-1
  nfs:
    path: /mnt/nfs_share/es_data_1
    server: 192.168.87.245
    readOnly: false
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-master
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: logging
    name: data-elasticsearch-master-0
  nfs:
    path: /mnt/nfs_share/es_master
    server: 192.168.87.245
    readOnly: false
EOF


cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-master-1
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: logging
    name: data-elasticsearch-master-1
  nfs:
    path: /mnt/nfs_share/es_master_1
    server: 192.168.87.245
    readOnly: false
EOF


cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: es-master-2
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: logging
    name: data-elasticsearch-master-2
  nfs:
    path: /mnt/nfs_share/es_master_2
    server: 192.168.87.245
    readOnly: false
EOF
```

### Install nfs-client for all nodes
```
sudo apt update
sudo apt install nfs-common
``` 

### install elasticsearch fluentd
```bash
kubectl create namespace logging
helm install elasticsearch stable/elasticsearch --namespace logging --set data.terminationGracePeriodSeconds=0
helm install fluentd --namespace logging stable/fluentd-elasticsearch --set elasticsearch.host=elasticsearch-client.logging.svc.cluster.local,elasticsearch.port=9200
helm install kibana --namespace logging stable/kibana --set env.ELASTICSEARCH_URL=http://elasticsearch-client.logging.svc.cluster.local:9200

kubectl -n logging get sts elasticsearch-data  -o yaml |sed 's/1536M/512M/g'| kubectl -n logging apply -f -
kubectl -n logging get sts elasticsearch-data  -o yaml |sed 's/1536m/256m/g'| kubectl -n logging apply -f -
kubectl -n logging get sts elasticsearch-master  -o yaml |sed 's/512M/512M/g'| kubectl -n logging apply -f -
kubectl -n logging get sts elasticsearch-master  -o yaml |sed 's/512m/256m/g'| kubectl -n logging apply -f -
kubectl -n logging get cm kibana -o yaml |sed 's/elasticsearch\:/elasticsearch-client\:/g'| kubectl -n logging apply -f -
kubectl -n logging delete pod -l  app=kibana
```

### create ingress
```bash
cat <<EOF | kubectl -n logging apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kibana
spec:
  rules:
  - host: kibana.192.168.87.245.nip.io
    http:
      paths:
      - backend:
          serviceName: kibana
          servicePort: 443
EOF
```


### TD: install nexus as repository for docker and 
### TD: install git
### TD: install authentication system
