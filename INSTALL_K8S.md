### Letting iptables see bridged traffic

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

### install docker
```bash
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```
```bash
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
```

### Installing kubeadm, kubelet and kubectl
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

### Disable Swap

```bash
sudo swapoff --all
sudo sed -i '/swap/d'  /etc/fstab
```
### Bring Up one node k8s
```bash
sudo kubeadm init
```
![Kube Init Result](kubeadm_init.png "Kube init result")

Run and save the environment variable
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Save the join node command
```
kubeadm join 192.168.87.245:6443 --token 8gtpuj.7a3m8y3dp3j5g9jo \
    --discovery-token-ca-cert-hash sha256:e847e6e43b17e2f3abee76738f6fa17612f8c3e8a2823fc392ddd4780892b88
	
```
### Virtual network (we choose calico)
```
kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

### Wait until all pods up and running
```
kubectl get pods --all-namespaces
```

![Get pod result](get_pod.png "Get pod result")

### Install ingress
```

kubectl create -f https://haproxy-ingress.github.io/resources/haproxy-ingress.yaml

#### nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/cloud/deploy.yaml


kubectl label node k8s-master role=ingress-controller

cat <<EOF | kubectl -n kubernetes-dashboard apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: dashboard
  annotations:
    ingress.kubernetes.io/secure-backends: "true"
spec:
  tls:
  - hosts:
      - 192.168.87.245.nip.io
  rules:
  - host: 192.168.87.245.nip.io
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
EOF


```
### Create a simple admin user
```
cat <<EOF | kubectl -n kubernetes-dashboard apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl -n kubernetes-dashboard apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```
### Get Token for login
```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

```
### Install a dashboard
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml


```
