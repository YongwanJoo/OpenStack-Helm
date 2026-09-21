# 구축 상태 매트릭스

이 문서는 2026-09-10까지의 회의록, 구축 기록, 트러블슈팅 문서에서 실제 성공 결과가 확인된 항목을 기준으로 작성했습니다. 계획이나 조사만 기록된 항목은 완료로 처리하지 않습니다.

| 영역 | 상태 | 완료 근거 |
| --- | --- | --- |
| OCI WireGuard Hub | 완료 | 노드의 허브-스포크 전환 및 VPN 통신 확인 |
| kubeadm Kubernetes | 완료 | Control Plane 초기화, Worker join, 노드 상태 확인 |
| Calico CNI | 완료 | VXLAN 기반 Pod 네트워크 및 cross-node 통신 확인 |
| Rook-Ceph | 완료 | MON/MGR 정상, 3 OSD, `HEALTH_OK` 확인 |
| Ceph CSI | 완료 | Controller/Node plugin 실행, MariaDB PVC `Bound` 확인 |
| MariaDB/RabbitMQ/Memcached | 완료 | Backend Pod 배포 및 의존 서비스 기동 확인 |
| Keystone | 완료 | Identity API 및 endpoint 사용 확인 |
| Glance | 완료 | Ceph 연동 PVC 및 Pod 실행 확인 |
| Cinder | 완료 | Ceph 상태와 연계한 Block Storage 서비스 실행 확인 |
| Placement | 완료 | Placement Pod 실행 확인 |
| Open vSwitch/Libvirt | 완료 | Compute 데이터 플레인 및 하이퍼바이저 연동 확인 |
| Nova | 완료 | Compute 노드 등록 및 VM 생성 확인 |
| Neutron Tenant Network | 완료 | DHCP, VM 간 통신, cross-compute VXLAN 확인 |
| External Network | 완료 | GRE over WireGuard, OCI NAT, VM 인터넷/DNS 확인 |
| Floating IP / 외부 SSH | 완료 | 공인 진입점에서 VM까지 inbound 연결 확인 |
| 재부팅 복구 | 완료 | OVS와 네트워크 경로 자동 복구 확인 |
| Horizon | 완료 | HTTPS 접근, 로그인, VM 조회 확인 |
| 장기 모니터링/성능 지표 | 조사·설계 | 병목과 metrics 필요성 논의만 확인됨 |
| 운영 자동화 확대 | 부분 완료 | 네트워크/OVS 복구는 자동화, 전체 Day-2 운영 자동화는 확인되지 않음 |

## 판정 원칙

- `완료`: 명령 실행 기록, 정상 상태, 실제 통신 결과 중 하나 이상이 문서에 명시된 경우
- `부분 완료`: 일부 자동화 또는 구성은 확인되지만 전체 범위의 종료 근거가 없는 경우
- `조사·설계`: 개념 정리, 비교, 개선안, 향후 논의만 있는 경우

이 매트릭스는 현재 저장소의 배포 보증서가 아니라 구축 당시의 검증 기록입니다. 재배포할 때는 각 단계 문서의 확인 명령으로 현재 상태를 다시 검증해야 합니다.
