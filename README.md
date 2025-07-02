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
# l2announcements.enabled : https://docs.cilium.io/en/stable/network/l2-announcements/
helm install cilium cilium/cilium --version 1.17.5 --namespace kube-system \
--set k8sServiceHost=auto --set k8sServicePort=6443 --set debug.enabled=true \
--set k8sServiceLookupConfigMapName=cluster-info --set k8sServiceLookupNamespace=kube-public \
--set rollOutCiliumPods=true --set routingMode=native --set autoDirectNodeRoutes=true \
--set bpf.masquerade=true --set bpf.hostRouting=true --set endpointRoutes.enabled=true \
--set ipam.mode=kubernetes --set k8s.requireIPv4PodCIDR=true --set kubeProxyReplacement=true \
--set ipv4NativeRoutingCIDR=10.0.0.0/16 --set installNoConntrackIptablesRules=true \
--set hubble.ui.enabled=true --set hubble.relay.enabled=true --set prometheus.enabled=true --set operator.prometheus.enabled=true --set hubble.metrics.enableOpenMetrics=true \
--set hubble.metrics.enabled="{dns:query;ignoreAAAA,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}" \
--set operator.replicas=1 --set l2announcements.enabled=true --set externalIPs.enabled=true
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

### 2. Metal MB 설치 (https://metallb.io/installation/) - Calico

#### 2.1 ARP 모드를 활성화

```bash
kubectl patch configmap kube-proxy -n kube-system --patch '{"data":{"config.conf":"apiVersion: kubeproxy.config.k8s.io/v1alpha1\nkind: KubeProxyConfiguration\nmode: \"ipvs\"\nipvs:\n  strictARP: true"}}'
```

#### 2.2 Meltal lb install

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

#### 2.3 IP Pool, L2 Layer 정의

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

### 3. Cilium IP Pool Setting (L2 Mode)

#### 3.1 ARP 모드를 활성화 : CiliumL2AnnouncementPolicy 생성

```yaml
# https://docs.cilium.io/en/stable/network/node-ipam/
# https://isovalent.com/blog/post/migrating-from-metallb-to-cilium/
# Cilium LoadBalancer IPAM 및 L2 서비스 공지 LAB : https://isovalent.com/labs/cilium-loadbalancer-ipam-and-l2-service-announcement/
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: cilium-l2-announcement-policy
spec:
  externalIPs: true
  loadBalancerIPs: true
```

#### 3.2 LB IP Pool 생성

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "cilium-ip-ip-pool"
spec:
  blocks:
    - cidr: "10.0.250.0/24"
```

- helm으로 설치한 후 옵션이 설정되지 않았을 때 하나씩 추가하는 방법

```bash
# cilium config set <KEY> <VALUE>
cilium config set l2announcements.enabled true
cilium config set externalIPs.enabled true
cilium config view
```

### 4. Ingress NginX Controller 생성

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```

### 5. ArgoCD install

#### 5.1 ArgoCD install

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type":"LoadBalancer"}}'
```

#### 5.2 ArgoCD CLI install

```bash
# Timeout으로 제대로 설치가 안될 경우 "--connect-timeout 30 --max-time 300" 옵션 추가
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

#### 5.3 ArgoCD Password check

```bash
argocd admin initial-password -n argocd
```

<!--

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
