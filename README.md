# OpenStack-Helm on Kubernetes

여러 대의 물리 장비와 OCI VM을 WireGuard VPN으로 연결하여 하나의 사설 네트워크를 구성하고, 그 위에 Kubernetes 및 OpenStack-Helm 환경을 구축하는 프로젝트입니다.

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

---

## Architecture

```text
                         Internet
                            │
                            ▼
                    ┌───────────────┐
                    │   WireGuard   │
                    │   VPN Hub     │
                    └───────┬───────┘
                            │
                     10.0.0.0/24
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
           Control Plane          Worker Nodes
                 │                     │
                 ▼                     ▼
              Helm ──────────► OpenStack-Helm
                                      │
                 ┌────────────────────┼────────────────────┐
                 │          │         │         │          │
              Keystone   Glance   Cinder   Placement    Nova
                                                        Neutron
```

### Kubernetes 내부 컴포넌트 구조

```text
                       Kubernetes
                           │
            ┌──────────────┼───────────────┐
            │              │               │
          Calico        MetalLB       Gateway API
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

## Deployment Guide

OpenStack-Helm을 기반으로 클러스터를 구축하고 각 컴포넌트를 배포하는 전체 과정을 0~5단계로 나누어 기록합니다.

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
Keystone, Glance, Cinder, Placement 등 핵심 서비스 배포 과정 및 향후 아젠다입니다.
→ **[상세 가이드 보기](docs/05-openstack-core.md)**

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
| [Troubleshooting](docs/troubleshooting/) | 인프라 구축 중 발생한 장애 트러블슈팅 일지 |

---

## Security

Repository에는 다음 정보를 포함하지 않습니다.

* WireGuard Private Key / Peer Secret
* SSH Private Key / Cloud API Key
* 실제 인증 정보 및 운영 환경의 민감한 정보

필요한 설정은 예시 파일(`.env.example`, `wg0.conf.example`)을 제공하고 실제 값은 별도로 관리합니다.
