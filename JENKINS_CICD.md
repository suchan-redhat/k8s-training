
### install jenkins plugins

Plugins needed
```
pipeline
Kubernetes CLI
pipeline
Docker pipeline
Git
Pipeline Utility Steps
```


### customize jenkins
```bash
mkdir jenkins_pipeline
vi jenkins_pipeline/Dockerfile
```


```
from jenkins/jenkins:2.235.1-lts-centos7
USER 0
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.0/bin/linux/amd64/kubectl ;\
    chmod +x ./kubectl ;\
    mv ./kubectl /usr/local/bin/kubectl ;\
    yum install -y buildah maven
```

Push the image to repo
```bash
sudo docker build -t myjenkins ./jenkins_pipeline 
sudo docker tag myjenkins tccsunny/myjenkins
sudo docker push tccsunny/myjenkins
```

### run and build jenkins 
```bash
kubectl create namespace cicd
kubectl -n cicd create deployment jenkins --image=tccsunny/myjenkins
kubectl -n cicd create service clusterip jenkins --tcp=8080:8080
```


### create ingress
```bash
cat <<EOF | kubectl -n cicd apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: cicd
spec:
  rules:
  - host: jenkins.192.168.87.245.nip.io
    http:
      paths:
      - backend:
          serviceName: jenkins
          servicePort: 8080
EOF
```

### create volume
```bash
k8s-master$ mkdir /mnt/nfs_share/jenkins_home
k8s-master$ chmod 0777 /mnt/nfs_share/jenkins_home
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: cicd
    name: jenkins
  nfs:
    path: /mnt/nfs_share/jenkins_home
    server: 192.168.87.245
    readOnly: false
EOF

cat <<EOF | kubectl -n cicd apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi
EOF


kubectl -n cicd patch deployment jenkins  -p '{"spec": { "template" : { "spec": { "containers": [{"name":"jenkins", "imagePullPolicy": "Always" , "securityContext":{"privileged": true},  "volumeMounts":[{"name":"jenkins","mountPath": "/var/jenkins_home"  }] }], "volumes": [ {"name": "jenkins", "persistentVolumeClaim": { "claimName":"jenkins" }} ]  } }  }}'

```


### initialize jenkins

```
http://jenkins.192.168.87.245.nip.io/
```

```bash
kubectl -n cicd exec -i -t jenkins-7c7584bfbc-7ch6n -- cat /var/jenkins_home/secrets/initialAdminPassword
```


### createing repo in GITHUB

### add and push your source code

```bash
cd springboot-jwt
mvn clean
echo "# demo_jwt" >> README.md
git init
git config --global user.email "sunnychan@tcc-consulting.com.hk"
git config --global user.name "Sunny Chan"
git add . -A
git commit -m "first commit"
git remote add origin https://github.com/sunny-tcc-consulting/demo_jwt.git
git push -u origin master

```

### exporting kubenetes resources
```bash
mkdir -p springboot-jwt/kubeResources
kubectl -n myapp get deployment,service,ingress -o yaml > springboot-jwt/kubeResources/kuberesource.yaml
cd springboot-jwt
git add . -A
git commit -m "first commit"
git push -u origin master
```

### Create credential - docker config

```
/root/.docker/config.json
```

### Create credential - kubeconfig
```
/home/k8s/.kube/config
```

### write jenkins file

```
node {
    stage('GIT Download') {
        git url: 'https://github.com/sunny-tcc-consulting/demo_jwt.git'
        println 'haha'
    }
    stage('compile and install') {
        sh 'mvn install'
    }
    
    stage('Image build and push') {
        withCredentials([file(credentialsId: 'dockerauth', variable: 'FILE')]) {
            sh 'export STORAGE_DRIVER=vfs; export BUILDAH_FORMAT=docker; buildah build-using-dockerfile --authfile $FILE -t docker://index.docker.io/tccsunny/myapp ./ '
        }
    }
    stage('Apply kubenetes Resource') {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'FILE')]) {
            def files = findFiles(glob: "kubeResources/**")
            for (file in files){
                if (! file.name.startsWith('.')) {
                    print file.path
                    sh """
                    kubectl --kubeconfig=$FILE -n myapp apply -f ${file.path}
                    """
                }
            }
        }
    }
}
```
