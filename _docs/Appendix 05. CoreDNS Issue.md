### 파드 내부에서 도메인 검색이 안될 때

파드내부에서 `/etc/resolv.conf` 파일 확인하면 coredns의 cluster ip 주소를 가리킨다. coredns의 로그를 살펴보면 아래와 같이 호스트의 네임서버로 통신이 되지 않고 차단되는 것을 확인할 수 있다.

```bash
$ kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
CoreDNS-1.11.3
linux/amd64, go1.21.11, a6338e9
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:44086->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:39416->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:34250->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:50215->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:60542->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:45561->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:59656->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:49368->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:40309->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 2880157511810145385.1418540156305842520. HINFO: read udp 10.100.1.240:57832->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 github.com. AAAA: read udp 10.100.1.240:38839->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 github.com. AAAA: read udp 10.100.1.240:52635->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 github.com. A: read udp 10.100.1.240:60467->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 github.com. A: read udp 10.100.1.240:58948->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 github.com. AAAA: read udp 10.100.1.240:56668->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 github.com. AAAA: read udp 10.100.1.240:46486->10.0.0.1:53: i/o timeout
[ERROR] plugin/errors: 2 github.com. A: read udp 10.100.1.240:38776->10.0.0.1:53: i/o timeout
```

#### 문제 추정 원인

- [Cilium이 설치되면 작업자 노드 DNS 확인이 실패합니다](https://github.com/cilium/cilium/issues/36406)

  - 쿠버네티스 1.31 설치 후 Cilium CNI 배포 시 DNS 검색을 실패하는 현상을 동일하게 겪는 케이스가 있음

  - 

- VirtualBox + Kubernetes 환경에서 자주 발생

- Pod 네트워크 → Host 네트워크 접근 시 NAT 경로 문제

- CNI 플러그인(flannel, calico 등)이 10.0.0.1 경로를 차단

- iptables 규칙이 Pod에서 gateway 접근 제한

#### 해결 방법

- coredns 설정 변경

```bash
kubectl edit configmap coredns -n kube-system
```

```yaml
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . 8.8.8.8 1.1.1.1    # 이 부분만 변경
        cache 30
        loop
        reload
        loadbalance
    }
```

```bash
kubectl rollout restart deployment/coredns -n kube-system
```

- 디버깅 파드 실행 후 nslookup으로 검색 여부 확인

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup github.com
```
