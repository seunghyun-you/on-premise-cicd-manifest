### 1. base image setting
- tools install
```bash
apt update -y
apt install -y openssh-server curl tree vim
```

- ip setting
```bash
vim /etc/netplan/50-cloud-init.yaml
netplan apply
```

- pem key setting
```bash
ssh-keygen -t rsa -b 2048 -f ubuntu.pem
```

---

### 2. master, node clone setting
- ip setting
master	10.0.0.10	127.0.0.1	8022
node01	10.0.0.11	127.0.0.1	8023
node02	10.0.0.12	127.0.0.1	8024

- lvm extend
```bash
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

- partition exten
```bash
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

---

### 3. kubespary (kuberbetes install)
- hosts
```bash
# /etc/hosts
10.0.0.10 master.example.com master
10.0.0.11 master.example.com node01
10.0.0.12 master.example.com node02
```

- pip install
```bash
sudo apt update -y
sudo apt install python3-pip -y
```

- kubespray install
```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
git checkout -b v2.26
sudo pip install -r requirements.txt --break-system-packages --ignore-installed
```

- configure cluster information
```bash
cp -rfp inventory/sample/ inventory/cluster
export ANSIBLE_PRIVATE_KEY_FILE=~/.ssh/ubuntu.pem
```

- ini configuration
```bash
# inventory/cluster/inventory.ini
[all]
master ansible_host=10.0.0.10   ip=10.0.0.10  etcd_member_name=etcd1
node01 ansible_host=10.0.0.11  ip=10.0.0.11
node02 ansible_host=10.0.0.12  ip=10.0.0.12

[kube_control_plane]
master

[etcd]
master

[kube_node]
node01
node02

[calico_crr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_crr
```

- k8s-cluster.yml 파일에 버전 지정
```bash
# inventory/cluster/group_vars/k8s_cluster/k8s-cluster.yml
kube_version: 1.30.4
```

- etcd.yml 파일에 버전 지정
```bash
# inventory/cluster/group_vars/etcd.yml
etcd_version: 3.5.12
```

- kubernetes install
```bash
ansible-playbook -i inventory/cluster/inventory.ini -v --become  --become-user=root cluster.yml
```

---

### 4. metal lb install
(https://metallb.io/installation/)

- ARP 모드를 활성화
```bash
$ kubectl edit configmap -n kube-system kube-proxy
>>  strictARP: true
```

- Meltal lb install
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

- define ip pool & l2 layer
```yaml
# metal-lb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: metal-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.0.0.100-10.0.0.110
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: metal-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - metal-ip-pool
```

- nginx ingress controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

---

### 5. private registry
- docker install
```bash
apt update -y
apt install docker.io -y
systemctl start docker
usermod -a -G docker $USER
chown root:docker /var/run/docker.sock
```

- nexus start in container
```bash
cd ~
mkdir nexus-directory
sudo chown -R 200 nexus-directory
docker run -d -p 8081:8081 -p 8082:8082  --name nexus -v ~/nexus-directory:/nexus-data sonatype/nexus3:latest
```

- nexus password check
```bash
docker exec -it nexus /bin/bash
cat /nexus-data/admin.password

- nexus access test (in nexkins server)
vim /etc/docker/daemon.json
{
	"insecure-registries" : [ "10.0.0.20:8082" ]
}
systemctl restart docker
docker login -u admin 10.0.0.20:8082
```

---

### 6. node server private registry sign

- containerd install
```bash
apt install containerd -y
```

- containerd setting
```bash
# /etc/containerd/config.toml
version = 2   # version 2로 수정
...
# plugins 항목 하단에 아래 내용 추가
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."10.0.0.20:8082"]
      endpoint = ["http://10.0.0.20:8082"]
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."10.0.0.20:8082".tls]
      insecure_skip_verify = true
```

---

### 7. jenkins install

- jenkins install
```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt install openjdk-17-jdk -y
sudo apt install jenkins maven -y
```

- jenkins start
```bash
systemctl enable jenkins
systemctl start jenkins
```

- jenkins setting
```bash
usermod -a -G docker jenkins
chmod 666 /var/run/docker.sock
```

- jenkins password check
```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

- jenkins recomanded plugin install

- jenkins additional plugin install
  - Docker Pipeline
  - Docker Commons
  - Pipeline: Stage View
  - Build Timestamp

---

### 8. jenkins settings 

- SCM SSH setting (Jenkins GitHub Connection)
(SCM SSH  : https://10cheon00.tistory.com/4)

- Nexus and GitHub Credential asign

- jenkins pipeline setting
(file     : cicd_sample/node-app/Jenkinsfile)
(handbook : https://www.jenkins.io/doc/book/getting-started/)


### 9. ArgoCD install

- ArgoCD install
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type":"LoadBalancer"}}'
```

- ArgoCD CLI install
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

- ArgoCD Password check
```bash
argocd admin initial-password -n argocd
```

---

### 10. helm 

- helm install
```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

---

### 11. DB Images 

- create docker file (postgresql 16.5)
```bash
docker build -t 10.0.0.20:8082/db:v1.0.0
docker push 10.0.0.20:8082/db:v1.0.0
```

- openebs install
```bash
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
kubectl patch sc -n openebs openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

- create manifest

- kubernetes resource test
```bash
ka statefuleset-db.yaml
ka secret-db.yaml
ka service-db.yaml
```

- access test
```bash
k exec -it sts-postgres-db-0 -- su - postgres -c psql
\l
# node table 생성 여부 확인
```



### 12. Argo CD Webhook Setting
- https://dobby-isfree.tistory.com/216