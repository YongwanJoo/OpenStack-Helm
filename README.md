# OpenStack-Helm on Kubernetes

여러 대의 물리 장비와 OCI VM을 WireGuard VPN으로 연결하여 하나의 사설 네트워크를 구성하고, 그 위에 Kubernetes 및 OpenStack-Helm 환경을 구축하는 프로젝트입니다.

> 문서 기준 시점: 2026-09-10. Kubernetes, Rook-Ceph, OpenStack 제어 서비스, Nova/Neutron, 외부 네트워크, Horizon까지 통합 검증했습니다.

---

## 팀원 소개

<div align="center">
  <table>
    <tr>
      <td align="center" width="200px">
        <a href="https://github.com/YongwanJoo">
          <img src="https://github.com/YongwanJoo.png" width="150px" alt="주용완"/>
        </a><br />
        <br />
        <b>주용완</b><br />
        &nbsp;<br />
        <a href="https://github.com/YongwanJoo">
          <img src="https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=GitHub&logoColor=white" alt="GitHub"/>
        </a>
      </td>
      <td align="center" width="200px">
        <a href="https://github.com/Smallt0wn">
          <img src="https://github.com/Smallt0wn.png" width="150px" alt="정장우"/>
        </a><br />
        <br />
        <b>정장우</b><br />
        &nbsp;<br />
        <a href="https://github.com/Smallt0wn">
          <img src="https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=GitHub&logoColor=white" alt="GitHub"/>
        </a>
      </td>
      <td align="center" width="200px">
        <a href="https://github.com/llokr1">
          <img src="https://github.com/llokr1.png" width="150px" alt="홍진기"/>
        </a><br />
        <br />
        <b>홍진기</b><br />
        &nbsp;<br />
        <a href="https://github.com/llokr1">
          <img src="https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=GitHub&logoColor=white" alt="GitHub"/>
        </a>
      </td>
    </tr>
  </table>
</div>

### 역할 및 기여 범위

| 팀원 | 확인된 주요 기여 |
| --- | --- |
| 주용완 | OpenStack/OpenStack-Helm 및 kubeadm 사전 조사, 설치 전 공통 구성과 초기 Helm 트러블슈팅 문서화, README/설치 문서 정리 |
| 정장우 | OCI/WireGuard 및 Kubernetes 구성, Rook-Ceph와 OpenStack 서비스 배포, Nova/Neutron/외부 네트워크/Horizon 통합 검증 |
| 홍진기 | OpenStack-Helm 배치 설계, Ceph 네트워크 장애 및 MariaDB PVC/Ceph-CSI 장애 분석·문서화 |

세부 분류와 근거는 [기여 근거](docs/contribution-evidence.md)에 정리했습니다. 팀 단위 결과와 개인 수행 기록을 구분하며, 기록으로 확인되지 않는 내용은 개인 기여로 확정하지 않습니다.

---

## Architecture

```text
                    External Client
                            │ SSH / HTTPS / API
                            ▼
                    ┌───────────────┐
                    │ OCI Public IP │
                    │ HAProxy / NAT │
                    └───────┬───────┘
                            │
                    GRE over WireGuard
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
        ▼                   ▼                    ▼
   Physical Nodes      Physical Nodes        OCI VM
   (Mini PC ×3)        (GRAM, Desktop)       (#1~#3)
        │                   │                    │
        └───────────────────┼────────────────────┘
                            │
                    Kubernetes Cluster
                            │
                 ┌──────────┴──────────┐
                 │                     │
           Control Plane       Compute / Storage Nodes
                 │                     │
                 ▼                     ▼
              Helm ──────────► OpenStack-Helm
                                      │
                 ┌────────────────────┼────────────────────┐
                 │          │         │         │          │
              Keystone   Glance   Cinder   Placement    Nova
                                                        Neutron
                                                           │
                                                    Tenant / External
                                                           │
                                                      OpenStack VM
```

### Kubernetes 내부 컴포넌트 구조

```text
                       Kubernetes
                           │
            ┌──────────────┼───────────────┐
            │              │               │
          Calico      Rook / Ceph      Gateway API
                                           │
                                      Envoy Gateway
                                           │
                                           ▼
                                  OpenStack Endpoint
                                           │
                    ┌──────────────────────┴─────────────────┐
                    │                                        │
              OpenStack-Helm                           Ceph / Rook
                    │                                        │
       ┌────────────┼─────────────┐                          │
       │            │             │                          │
   RabbitMQ      MariaDB      Memcached                      │
       │            │             │                          │
       └────────────┼─────────────┘                          │
                    │                                        │
                 Keystone                                    │
                    │                                        │
          ┌─────────┼──────────┐                             │
          │         │          │                             │
       Glance   Placement    Neutron                         │
          │         │          │                             │
          │         │      Open vSwitch                      │
          │         │          │                             │
          │       Nova─────────┘                             │
          │         │                                        │
          │      Libvirt                                     │
          │         │                                        │
          └─────────┼────────────────────────────────────────┘
                    │
                 QEMU/KVM
                    │
                    ▼
               OpenStack VM
```

### Network

모든 노드는 WireGuard Hub-and-Spoke 구조를 통해 사설 네트워크로 연결합니다.

| 항목              | 구성            |
| --------------- | ------------- |
| VPN             | WireGuard     |
| Private Network | `10.0.0.0/24` |
| CNI             | Calico (VXLAN)|
| Node Count      | 8             |

> 실제 Public IP 및 WireGuard 인증 정보는 보안상의 이유로 Repository에 기록하지 않습니다.

---

## Hardware Nodes

### Physical Nodes

| Node      | 소유자 | CPU                       |  RAM |   Storage | 용도                       | Private IP  |
| --------- | --- | ------------------------- | ---: | --------: | ------------------------ | ----------- |
| Mini PC   | 진기  | AMD Ryzen 7               | 16GB | SSD 512GB | -                        | `10.0.0.1`  |
| Mini PC   | 장우  | AMD Ryzen 7               | 16GB | SSD 512GB | Kubernetes Control Plane | `10.0.0.7`  |
| GRAM      | 장우  | Intel Core i5             | 16GB | SSD 512GB | -                        | `10.0.0.6`  |
| Desktop 1 | 진기  | Intel Core i3             |  8GB |   HDD 1TB | -                        | `10.0.0.5`  |
| Mini PC   | 용완  | AMD Ryzen 7 H255 (8C/12T) | 30GB |   SSD 1TB | -                        | `10.0.0.10` |

### OCI Nodes

| Node      | 소유자 | CPU         | RAM | Storage | Private IP  |
| --------- | --- | ----------- | --: | ------: | ----------- |
| OCI VM #1 | 장우  | vCPU 1 Core | 1GB |   100GB | `10.0.0.8`  |
| OCI VM #2 | 장우  | vCPU 1 Core | 1GB |   100GB | `10.0.0.9`  |
| OCI VM #3 | 용완  | vCPU 1 Core | 1GB |   100GB | `10.0.0.11` |

### Resource Summary

| 구분 | 노드 수 | 총 RAM |
| --- | ---: | ---: |
| Physical | 5 | 76GB |
| OCI VM | 3 | 3GB |
| **합계** | **8** | **79GB** |

> 실제 사용 가능한 CPU 및 메모리는 VM 할당량, 운영체제, Kubernetes 시스템 Pod 등의 리소스 사용량에 따라 달라집니다.

---

## 현재 구축 완료 범위

| 영역 | 상태 | 검증 내용 |
| --- | --- | --- |
| WireGuard / OCI Hub | 완료 | 노드 간 VPN 통신 및 허브-스포크 경로 |
| Kubernetes / Calico | 완료 | kubeadm v1.33.13 클러스터 및 Calico VXLAN |
| Rook-Ceph / CSI | 완료 | 3 OSD `HEALTH_OK`, RBD PVC 동적 프로비저닝 |
| OpenStack Backend | 완료 | MariaDB, RabbitMQ, Memcached |
| OpenStack Core | 완료 | Keystone, Glance, Cinder, Placement |
| Compute / Network | 완료 | Open vSwitch, Libvirt, Nova, Neutron |
| Tenant Network | 완료 | DHCP, 서로 다른 Compute 간 VM 통신 |
| External Network | 완료 | GRE over WireGuard, OCI NAT, Floating IP, 외부 SSH |
| 운영 복구 | 완료 | 재부팅 후 OVS 재연결 및 경로 복구 |
| Horizon | 완료 | HTTPS 접속, 로그인, 인스턴스 조회 |

상세 상태와 판정 기준은 [구축 상태 매트릭스](docs/status-matrix.md)를 참고합니다.

---

## Deployment Guide

OpenStack-Helm을 기반으로 클러스터를 구축하고 각 컴포넌트를 배포·검증하는 전체 과정을 0~8단계로 기록합니다.

### 0. Kubernetes & WireGuard 설정
사전에 모든 노드를 WireGuard VPN(`10.0.0.0/24`)으로 연결하고, kubeadm을 통해 클러스터(v1.33.13) 및 Calico CNI 인프라를 구성합니다.
→ **[상세 가이드 보기](docs/00-kubernetes-wireguard.md)**

### 1. OpenStack-Helm 기반 구성 요소 설치
Helm Plugin(`osh`) 설치 및 GitOps 기반 디렉토리 아키텍처 환경 설정 과정입니다.
→ **[상세 가이드 보기](docs/01-openstack-helm-base.md)**

### 2. 네트워크 설정 (MetalLB & Envoy Gateway)
온프레미스 환경에서 외부 통신을 담당하는 LoadBalancer(MetalLB) 및 트래픽 라우팅을 위한 Gateway API(Envoy) 구성 과정입니다.
→ **[상세 가이드 보기](docs/02-network-setup.md)**

### 3. 물리 저장공간 설정 (Ceph & Rook)
클러스터 전반에 걸쳐 사용할 분산 스토리지 백엔드(Ceph)와 이를 관리하는 오퍼레이터(Rook) 배포 과정입니다.
→ **[상세 가이드 보기](docs/03-storage-setup.md)**

### 4. OpenStack-Helm Backend 설치
OpenStack 서비스 간 데이터 저장과 메시지 통신을 위한 기반 서비스(MariaDB, RabbitMQ, Memcached) 배포 과정입니다.
→ **[상세 가이드 보기](docs/04-openstack-backend.md)**

### 5. OpenStack-Helm 핵심 요소 설치
Keystone, Glance, Cinder, Placement 등 핵심 서비스 배포 과정입니다.
→ **[상세 가이드 보기](docs/05-openstack-core.md)**

### 6. Compute & Network 서비스
Open vSwitch, Libvirt, Nova, Neutron을 배포하고 Compute 노드 및 데이터 플레인을 검증합니다.
→ **[상세 가이드 보기](docs/06-openstack-compute-network.md)**

### 7. Tenant & External Network
Tenant 네트워크부터 GRE over WireGuard, OCI NAT, Floating IP, 외부 SSH까지의 최종 경로를 기록합니다.
→ **[상세 가이드 보기](docs/07-tenant-external-network.md)**

### 8. Horizon
Horizon 배포와 HTTPS 접근, 로그인 및 인스턴스 조회 검증 과정을 기록합니다.
→ **[상세 가이드 보기](docs/08-horizon.md)**

---

## Documentation & Guides

| 문서 | 설명 |
| --- | --- |
| [0. Kubernetes & WireGuard](docs/00-kubernetes-wireguard.md) | WireGuard VPN 구성 및 kubeadm 기반 클러스터 설치 기록 |
| [1. OpenStack-Helm Base](docs/01-openstack-helm-base.md) | Helm Plugin 등 배포 초기 인프라 셋업 기록 |
| [2. Network Setup](docs/02-network-setup.md) | MetalLB & Envoy Gateway 외부 트래픽 라우팅 설정 가이드 |
| [3. Storage Setup](docs/03-storage-setup.md) | 분산 스토리지 백엔드(Ceph/Rook) 아키텍처 및 배포 |
| [4. OpenStack Backend](docs/04-openstack-backend.md) | DB 및 MQ 인프라(MariaDB, RabbitMQ, Memcached) 설정 |
| [5. OpenStack Core](docs/05-openstack-core.md) | OpenStack 핵심 서비스(Keystone, Glance 등) 배포 |
| [6. Compute & Network](docs/06-openstack-compute-network.md) | Open vSwitch, Libvirt, Nova, Neutron 배포와 검증 |
| [7. Tenant & External Network](docs/07-tenant-external-network.md) | VM 네트워크, OCI NAT, Floating IP 및 외부 SSH 검증 |
| [8. Horizon](docs/08-horizon.md) | Horizon HTTPS 접근과 대시보드 검증 |
| [구축 상태 매트릭스](docs/status-matrix.md) | 실제 완료·부분 완료·조사 항목 구분 |
| [기여 근거](docs/contribution-evidence.md) | 회의록, 작업 문서, Git 커밋 기반 기여 분류 |
| [Troubleshooting](docs/troubleshooting/) | 인프라 구축 중 발생한 장애 트러블슈팅 일지 |

---

## Security

Repository에는 다음 정보를 포함하지 않습니다.

* WireGuard Private Key / Peer Secret
* SSH Private Key / Cloud API Key
* 실제 인증 정보 및 운영 환경의 민감한 정보
* 공인 IP, 내부 관리 주소, 개인 계정 및 호스트별 접속 정보

필요한 설정은 예시 파일(`.env.example`, `wg0.conf.example`)을 제공하고 실제 값은 별도로 관리합니다.
