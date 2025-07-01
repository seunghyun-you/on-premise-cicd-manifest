## 쿠버네티스 Add-on 설치

### 1. CNI 구성

#### 1.1 Cilium Install

#### ① helm 설치

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

#### ② Cilium Helm Repository 추가

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

#### ③ Cilium Install

```bash
# helm option : https://docs.cilium.io/en/stable/helm-reference/
# routingMode : https://docs.cilium.io/en/stable/network/concepts/routing/
# ipam.mode : https://docs.cilium.io/en/stable/network/concepts/ipam/
# ipv4NativeRoutingCIDR : https://docs.cilium.io/en/stable/network/concepts/masquerading/
helm install cilium cilium/cilium --version 1.17.5 --namespace kube-system \
--set k8sServiceHost=auto --set k8sServicePort=6443 --set debug.enabled=true \
--set k8sServiceLookupConfigMapName=cluster-info --set k8sServiceLookupNamespace=kube-public \
--set rollOutCiliumPods=true --set routingMode=native --set autoDirectNodeRoutes=true \
--set bpf.masquerade=true --set bpf.hostRouting=true --set endpointRoutes.enabled=true \
--set ipam.mode=kubernetes --set k8s.requireIPv4PodCIDR=true --set kubeProxyReplacement=true \
--set ipv4NativeRoutingCIDR=10.0.0.0/16 --set installNoConntrackIptablesRules=true \
--set hubble.ui.enabled=true --set hubble.relay.enabled=true --set prometheus.enabled=true --set operator.prometheus.enabled=true --set hubble.metrics.enableOpenMetrics=true \
--set hubble.metrics.enabled="{dns:query;ignoreAAAA,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}" \
--set operator.replicas=1
```

#### ④ Cilium CLI 설치

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

```bash
cilium status --wait
```

#### ⑤ Cluster 상태 확인

```bash
kubectl get po -A
```

#### 1.2 Calico CNI 설치

```bash
# https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/tigera-operator.yaml
```

<!--

### 3. Install Kubernetes Cluster (with kubespary)

#### 3.1 pip 설치

```bash
sudo apt update -y
sudo apt install python3-pip -y
```

#### 3.2 kubespray install

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
sudo pip install -r requirements.txt --break-system-packages --ignore-installed
```

#### 3.3 configure cluster information (calico cni version)

```bash
cp -rfp inventory/sample/ inventory/cluster
```

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

#### 3.4 kubernetes 버전 지정

```bash
# inventory/cluster/group_vars/k8s_cluster/k8s-cluster.yml
kube_version: 1.30.4
```

#### 3.5 etcd 버전 지정

```bash
# inventory/cluster/group_vars/etcd.yml
etcd_version: 3.5.12
```

#### 3.6 ssh pem key 경로 설정

```bash
export ANSIBLE_PRIVATE_KEY_FILE=~/.ssh/ubuntu.pem
```

#### 3.7 install kubernetes cluster

```bash
ansible-playbook -i inventory/cluster/inventory.ini -v --become-user=root cluster.yml
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
docker run -d -p 8081:8081 -p 8082:8082  --name nexus --restart=always -v ~/nexus-directory:/nexus-data sonatype/nexus3:latest
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
  (SCM SSH : https://10cheon00.tistory.com/4)

- Nexus and GitHub Credential asign

- jenkins pipeline setting
  (file : cicd_sample/node-app/Jenkinsfile)
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

- https://dobby-isfree.tistory.com/216 -->
