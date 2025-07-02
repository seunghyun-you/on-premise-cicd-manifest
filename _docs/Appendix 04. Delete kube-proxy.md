### kube-proxy 관련 리소스 삭제

#### 데몬셋 삭제

```bash
kubectl delete daemonset kube-proxy -n kube-system
```

#### 컨피그맵 삭제

```bash
kubectl delete configmap kube-proxy -n kube-system
```

#### 서비스어카운트 삭제 (필요시)

```bash
kubectl delete serviceaccount kube-proxy -n kube-system
```

#### kube-proxy가 생성한 iptables 규칙 확인

```bash
sudo iptables -t nat -L | grep KUBE
```

#### 필요시 iptables 규칙 플러시 (주의: 다른 규칙도 삭제됨)

```bash
sudo iptables -t nat -F
sudo iptables -t filter -F
sudo iptables -t mangle -F
```
