## Nat Network

### Nat Network Restart

Host PC에서 VirtualBox의 VM은 실행중인 상태이지만 VM으로 접속이 안되는 경우 Nat Network Router의 문제가 발생한 경우로 재실행해야 한다.

```bash
VBoxManage natnetwork stop --netname Kubernetes
VBoxManage natnetwork start --netname Kubernetes
```

### Nat Network Portforward setting

- 참고 문서 : https://docs.oracle.com/en/virtualization/virtualbox/7.1/user/networkingdetails.html#network_nat_service

- 신규 포워딩 정책 추가

```bash
VBoxManage natnetwork modify \
  --netname Kubernetes --port-forward-4 "web_argocd_swagger:tcp:[192.168.200.2]:18888:[10.0.250.0]:8080"
```

- 포워딩 정책 삭제

```bash
VBoxManage natnetwork modify --netname natnet1 --port-forward-4 delete ssh
```

## Virtual Box VM Managing

### VM 목록 보기

- 기본 조회 명령

```bash
VBoxManage list vms              # 모든 VM 목록
VBoxManage list runningvms       # 실행 중인 VM만 보기
VBoxManage list vms --long       # 상세 정보 포함
```

- On/Off 상태만 확인하는 명령

```bash
vms=$(VBoxManage list vms | awk '{print $1}' | sed 's/"//g')
n=1
for vm in $vms; do
    if VBoxManage list runningvms | grep -q "\"$vm\""; then
        status="On"
    else
        status="Off"
    fi
    echo "$n. $vm Status: $status"
    n=$((n+1))
done
```

### VM 상태 확인

```bash
VBoxManage showvminfo "VM_NAME" | grep State
VBoxManage showvminfo "VM_NAME" --machinereadable | grep VMState
```

### VM 실행

```bash
VBoxManage startvm "VM_NAME"                    # GUI로 실행
VBoxManage startvm "VM_NAME" --type headless    # 백그라운드 실행 (GUI 없이)
VBoxManage startvm "VM_NAME" --type gui         # GUI로 실행
```

### VM 종료

```bash
VBoxManage controlvm "VM_NAME" poweroff          # 강제 종료 (전원 버튼)
VBoxManage controlvm "VM_NAME" acpipowerbutton   # 정상 종료 (ACPI 신호)
VBoxManage controlvm "VM_NAME" savestate         # 상태 저장 후 종료
```

### VM 일시정지

```bash
VBoxManage controlvm "VM_NAME" pause             # 일시정지
VBoxManage controlvm "VM_NAME" resume            # 재개
```
