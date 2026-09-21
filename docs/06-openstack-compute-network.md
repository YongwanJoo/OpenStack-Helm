# 6단계: OpenStack Compute & Network

Keystone, Glance, Cinder, Placement 배포 이후 Open vSwitch, Libvirt, Nova, Neutron을 연결해 실제 VM을 실행하는 단계입니다. 이 문서는 최종 구축 기록을 기준으로 완료 상태와 검증 지점을 정리합니다.

## 1. 배포 범위

| 구성 요소 | 역할 | 최종 상태 |
| --- | --- | --- |
| Open vSwitch | VM 가상 스위치 및 터널 데이터 플레인 | 완료 |
| Libvirt | QEMU/KVM 하이퍼바이저 관리 | 완료 |
| Nova | VM 스케줄링 및 Compute 서비스 | 완료 |
| Neutron | Tenant 네트워크, DHCP, Router, Floating IP | 완료 |

버전별 기본값은 `manifests/overrides/` 아래의 `openvswitch`, `libvirt`, `nova`, `neutron` 파일로 관리합니다. 클러스터별 주소, 인증값, 키는 저장소에 커밋하지 않습니다.

## 2. 노드 역할

- KVM을 사용할 수 있는 노드만 Nova Compute 역할을 맡습니다.
- 스토리지 전용 노드는 Ceph에는 참여할 수 있지만 KVM을 지원하지 않으면 Compute 대상에서 제외합니다.
- OVS 터널 인터페이스는 노드 간 연결이 보장된 WireGuard 인터페이스를 사용합니다.

노드 라벨과 실제 하이퍼바이저 등록 결과를 함께 확인합니다.

```bash
kubectl get nodes --show-labels
openstack hypervisor list
openstack compute service list
```

## 3. 배포 순서

최종 구축에서는 다음 의존성 순서로 진행했습니다.

```text
Open vSwitch -> Libvirt -> Nova -> Neutron
```

각 단계에서 다음 단계로 넘어가기 전에 Helm release와 Pod가 모두 준비됐는지 확인합니다.

```bash
helm list -n openstack
kubectl get pods -n openstack -o wide
helm osh wait-for-pods openstack
```

## 4. VXLAN 포트 분리

Calico VXLAN과 Neutron VXLAN이 같은 UDP 포트를 사용하면 패킷이 잘못된 overlay로 전달될 수 있습니다. 이 환경에서는 Calico와 충돌하지 않도록 Neutron VXLAN 포트를 `8472/udp`로 분리하고, 해당 포트를 Kubernetes 노드 사이에서 허용했습니다.

설정 변경 후 각 노드에서 OVS와 터널 포트를 확인합니다.

```bash
sudo ovs-vsctl show
sudo ss -lunp | grep 8472
```

장애 분석 과정은 [Neutron VXLAN 포트 충돌](troubleshooting/03-neutron-vxlan-port-firewall.md)에 정리했습니다.

## 5. 완료 검증

```bash
openstack network agent list
openstack hypervisor list
openstack server list --all-projects
```

최종 검증 기준은 다음과 같습니다.

- Nova Compute 서비스가 KVM 가능 노드에 등록된다.
- Neutron agent가 정상 상태로 조회된다.
- 서로 다른 Compute 노드에 생성한 VM이 DHCP 주소를 받는다.
- 같은 Tenant 네트워크의 VM 사이에 통신할 수 있다.

외부 네트워크와 Floating IP 검증은 [7단계](07-tenant-external-network.md)에서 이어집니다.
