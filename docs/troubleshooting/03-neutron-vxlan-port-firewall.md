# Neutron VXLAN 포트 충돌 및 방화벽

## 증상

- VM이 DHCP 주소를 받지 못하거나 서로 다른 Compute 노드의 VM끼리 통신하지 못함
- Nova 인스턴스는 실행 중이지만 Tenant 네트워크 패킷이 다른 노드로 전달되지 않음
- OVS 터널은 보이지만 VXLAN 패킷이 기대한 서비스에 도달하지 않음

## 원인

Calico와 Neutron이 같은 underlay인 WireGuard를 사용하면서 기본 VXLAN UDP 포트가 겹쳤습니다. 동일 포트의 overlay가 공존하면 패킷이 잘못 처리되거나 방화벽 규칙이 모호해질 수 있습니다.

## 조치

1. Calico VXLAN 포트와 현재 리스닝 상태를 확인합니다.
2. Neutron VXLAN 포트를 `8472/udp`로 분리합니다.
3. Compute/Network 노드 사이에서 해당 UDP 포트를 허용합니다.
4. Neutron agent와 OVS 관련 Pod를 재기동한 뒤 터널을 다시 확인합니다.

```bash
sudo ss -lunp | grep -E '4789|8472'
sudo ovs-vsctl show
openstack network agent list
```

방화벽 규칙은 실제 Compute/Network 노드의 VPN 주소 범위로 제한합니다. 저장소에는 호스트별 주소나 운영 규칙 원문을 커밋하지 않습니다.

## 검증

- 서로 다른 Compute 노드에 VM을 한 대씩 생성합니다.
- 두 VM이 DHCP 주소를 받는지 확인합니다.
- VM 사이 ping과 보안 그룹 허용 트래픽을 확인합니다.
- 노드에서 `8472/udp` 패킷이 송수신되는지 확인합니다.

최종 환경에서는 Neutron VXLAN을 `8472/udp`로 분리한 뒤 cross-compute VM 통신에 성공했습니다.
