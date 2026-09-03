# 5단계: OpenStack-Helm 핵심 요소 설치

기반 인프라(네트워크, 스토리지, DB, 메시지 큐 등)가 모두 안정화된 이후, 실제 클라우드 환경을 구성하는 OpenStack의 코어 서비스들을 배포하는 단계입니다. 각 서비스들은 서로 강한 의존성을 띠고 있으므로 순차적인 배포가 중요합니다. 배포 명령어는 OpenStack-Helm 공식 차트와 `get-values-overrides`로 가져온 설정값을 혼합하여 사용합니다.

## 1. Keystone (Identity) 배포

OpenStack의 인증 및 권한 인가(Authorization)를 전담하는 가장 핵심적인 서비스입니다. 모든 OpenStack 서비스는 구동될 때 Keystone에 자신의 Service Endpoint를 등록하며, 사용자 및 다른 서비스 간 통신 시 Keystone이 발급한 토큰을 이용해 권한을 검증합니다.

* **배포 유의사항**: 이 서비스가 가장 먼저 정상 배포되어야 이후 다른 서비스들이 Endpoint를 등록하고 동작할 수 있습니다. 

### 설치 명령어
```bash
helm upgrade --install keystone openstack-helm/keystone \
  --namespace openstack \
  -f manifests/overrides/keystone.yaml \
  -f manifests/custom/keystone.yaml
```

배포 완료 후, Keystone API가 정상 응답하는지 파드 상태를 점검합니다.
```bash
helm osh wait-for-pods openstack
```

## 2. Glance (Image) 배포

가상머신을 띄우기 위한 OS 이미지(QCOW2, RAW, ISO 등)를 등록하고 관리하는 저장소 서비스입니다.

* **스토리지 연동**: 3단계에서 구성한 **Ceph 스토리지(RBD)**를 백엔드로 사용합니다. Ceph PVC(`general` StorageClass)를 할당받아 이미지를 분산 저장함으로써 고가용성과 빠른 I/O 속도를 확보합니다.

### 설치 명령어
```bash
helm upgrade --install glance openstack-helm/glance \
  --namespace openstack \
  -f manifests/overrides/glance.yaml \
  -f manifests/custom/glance.yaml
```

## 3. Cinder (Block Storage) 배포

가상머신(인스턴스) 운영 중에 추가로 부착할 수 있는 블록(디스크) 볼륨을 제공하는 서비스입니다. AWS의 EBS 역할을 합니다.

* **스토리지 연동**: Glance와 마찬가지로 Ceph RBD API와 직접 통신하도록 구성됩니다. Ceph 내에 `cinder.volumes`라는 전용 Pool을 프로비저닝하여 볼륨 데이터를 안전하게 보관합니다.

### 설치 명령어
```bash
helm upgrade --install cinder openstack-helm/cinder \
  --namespace openstack \
  -f manifests/overrides/cinder.yaml \
  -f manifests/custom/cinder.yaml
```

## 4. Placement (Resource) 배포

클러스터 내의 물리 자원(Compute Node의 CPU, RAM, 스토리지 여유분 등) 인벤토리를 추적하고 할당량을 관리하는 서비스입니다. Nova 스케줄러가 어느 노드에 VM을 생성할지 결정할 때 Placement API를 참조하여 최적의 노드를 찾습니다.

### 설치 명령어
```bash
helm upgrade --install placement openstack-helm/placement \
  --namespace openstack \
  -f manifests/overrides/placement.yaml \
  -f manifests/custom/placement.yaml
```

---

## 5. 공식 레퍼런스
* [OpenStack-Helm Core Deployment Guide](https://docs.openstack.org/openstack-helm/latest/install/openstack.html)

## 향후 진행할 핵심 구성 (Next Steps)

현재 제어 영역(Control Plane) API 구성이 대부분 마무리된 상태입니다. 이후 실제 가상머신을 구동하기 위해 다음 단계들을 순차적으로 진행할 예정입니다.

1. **네트워크 데이터 플레인 배포 (Open vSwitch → Neutron)**
   * 컴퓨트 노드 간 통신(WireGuard `wg0`)과 외부 통신(Provider Network, `br-ex`)을 위한 물리 NIC 맵핑 구성
   * 가상 네트워크 스위치 및 라우팅 인프라 구성
2. **컴퓨트 노드 배포 (Libvirt → Nova)**
   * 하이퍼바이저(KVM/QEMU)를 관리하는 Libvirt 데몬 배포
   * Compute 노드 자원을 등록하고 인스턴스 스케줄링을 담당할 Nova 배포
3. **사용자 인터페이스 (Horizon)**
   * 구축된 모든 자원을 쉽게 관리하기 위한 웹 대시보드(Horizon) 배포
   * MetalLB IPAddressPool 연동 및 Envoy Gateway HTTPRoute 구성을 통한 외부 공인 트래픽 라우팅
