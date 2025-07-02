### Nat Network Restart

Host PC에서 VirtualBox의 VM은 실행중인 상태이지만 VM으로 접속이 안되는 경우 Nat Network Router의 문제가 발생한 경우로 재실행해야 한다.

```bash
VBoxManage natnetwork stop --netname Kubernetes
VBoxManage natnetwork start --netname Kubernetes
```
