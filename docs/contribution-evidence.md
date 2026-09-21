# 기여 근거

개인 기여와 팀 결과를 혼동하지 않기 위해 회의록, 개인 작업 문서, Git 커밋에서 확인되는 범위만 정리했습니다. 동일 작업에 여러 기록이 있으면 개인 주도와 팀 결과를 함께 표기합니다.

## 주용완

| 분류 | 확인된 내용 | 근거 |
| --- | --- | --- |
| 직접 수행 | OpenStack/OpenStack-Helm 구조 및 kubeadm/container runtime 조사·문서화 | 개인 문서 `OpenStack이란?`, `OpenStack-Helm 간략하게`, `Kubeadm 기반 Kubernetes 구축`, `Container Runtime 정리` |
| 직접 수행 | Git, Helm, OSH plugin, source clone, node label 등 설치 전 구성과 prep script 정리 | 개인 문서 `OpenStack Helm 전 구성` |
| 직접 수행 | Helm release 재사용, StorageClass/PVC Pending, helm-toolkit dependency 등 초기 실습 장애 해결 | 개인 트러블슈팅 문서 |
| 직접 수행 | 저장소 초기 README와 설치 문서 정리 | Git commits `docs: README.md & .gitignore 추가`, `docs: add install steps` |
| 조사·설계 | OpenStack-Helm 버전 관리, Rook/Ceph 관리, security/operator 관점 제안 | 8주차 회의록 |

Kubernetes Control Plane 전체 구축이나 최종 OpenStack 네트워크 배포를 주도했다는 근거는 확인되지 않아 개인 수행으로 분류하지 않습니다.

## 정장우

| 분류 | 확인된 내용 | 근거 |
| --- | --- | --- |
| 직접 수행 | OCI/WireGuard Hub, 노드 OS 및 연결 구성 | `OCI instance 세팅 과정 (1)/(2)` |
| 직접 수행 | kubeadm Control Plane, Worker join, Calico 구성 | `Kubeadm 설치 진행`, `k8s network setting`, `k8s worker node join`, 3~4주차 회의록 |
| 직접 수행 | Rook-Ceph OSD/네트워크 장애 해결 | `Ceph 트러블 슈팅 진행`, 6주차 회의록 |
| 직접 수행 | MariaDB/RabbitMQ/Memcached와 Keystone/Glance/Cinder/Placement 배포 | `Openstack 나머지 설치 과정_Be/Service` |
| 직접 수행 | OVS, Libvirt, Nova, Neutron, external network, Floating IP, Horizon 통합 | `네트워크 세팅 진행하기`, `네트워크 세팅 진행하기(2)` |
| 직접 수행 | overrides, infrastructure values, 관리 scripts와 2026.1 환경 표준화 | 2026-09-02 Git commits |

## 홍진기

| 분류 | 확인된 내용 | 근거 |
| --- | --- | --- |
| 직접 수행 | 노드/컴포넌트 배치와 OpenStack-Helm 설치 가이드 | 개인 설치 가이드, 4주차 회의록 |
| 직접 수행 | Ceph MGR CrashLoopBackOff의 UFW/WireGuard 경로 원인 분석 | Ceph MGR 트러블슈팅 문서, 6주차 회의록 |
| 직접 수행 | MariaDB PVC Pending의 Ceph RBD CSI ServiceAccount/RBAC 누락 분석 | MariaDB PVC 트러블슈팅 문서 |

## 팀 공동 결과

- OpenStack-Helm 버전과 배포 방향 논의
- Kubernetes/OpenStack-Helm 클러스터 구성 전략 결정
- Ceph, PVC, StorageClass, MariaDB 연동 문제 해결
- GitHub 문서 구조와 역할 논의
- Nova/Neutron, external network, Horizon까지 포함한 private cloud 최종 검증

최종 구축물은 팀 결과입니다. 개인별 표는 기록상 주도 또는 직접 수행이 확인되는 부분을 설명하며, 공동 논의와 다른 팀원의 작업을 개인 성과로 합산하지 않습니다.
