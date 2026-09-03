# 3단계: 물리 저장공간 설정 (Ceph & Rook)

OpenStack-Helm 인프라에서 가장 중요하고 복잡한 백엔드 중 하나인 **Ceph**와 이를 관리하는 **Rook**에 대한 배포 및 구성 과정입니다.

## 1. Ceph와 Rook의 역할

### Ceph란?
Ceph는 단일 분산 스토리지 클러스터에서 **Block, Object, File Storage**를 모두 제공하는 오픈소스 스토리지 플랫폼입니다. OpenStack 환경에서는 보통 LVM 대신 Ceph를 스토리지 백엔드로 사용하며, 다음과 같은 모든 핵심 저장소 역할을 도맡아 합니다.

* **Glance (Image Storage)**: 가상머신의 OS 이미지(QCOW2, RAW 등) 저장
* **Cinder (Block Storage)**: 가상머신에 부착되는 추가 볼륨(디스크) 저장
* **Nova (Compute Storage)**: 가상머신의 인스턴스 부트 디스크 상태 저장
* **Kubernetes PVC**: MariaDB, RabbitMQ 등 인프라 파드들이 사용하는 영구 볼륨 제공

### Rook이란?
Rook은 Kubernetes 환경에서 매우 복잡한 Ceph 데몬(MON, MGR, OSD 등)을 쉽게 배포, 구성, 관리, 업그레이드할 수 있게 해주는 **Storage Operator**입니다. 이를 통해 `CephCluster`라는 CRD 하나만 선언해도 내부적으로 쿼럼 유지, 디스크 프로비저닝 등을 알아서 스케줄링하고 관리해 줍니다.

## 2. 배포 및 설치 명령어

Rook과 Ceph를 배포하기 위해 공식 Helm 차트를 사용합니다.

### 2.1 Rook Operator 설치
먼저 Ceph 클러스터를 관리할 Rook Operator를 `rook-ceph` 네임스페이스에 설치합니다.

```bash
# Rook-Ceph Helm Repository 추가
helm repo add rook-release https://charts.rook.io/release
helm repo update

# Namespace 생성
kubectl create namespace rook-ceph

# Rook Operator 설치
helm install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph \
  --version v1.11.7
```

설치 후 Operator 파드가 정상적으로 Running 상태인지 확인합니다.
```bash
kubectl get pods -n rook-ceph
```

### 2.2 Ceph Cluster 및 OSD 구성
Operator가 준비되면 실제 스토리지 노드의 디스크를 묶어줄 `CephCluster`를 배포합니다.
주로 `values.yaml`을 수정하여 OSD로 사용할 디스크(예: `/dev/sdb`, `/dev/nvme0n1`)를 지정합니다.

```bash
# Ceph Cluster 차트 설치
helm install rook-ceph-cluster \
  --namespace rook-ceph \
  rook-release/rook-ceph-cluster \
  -f manifests/infrastructure/ceph-cluster-values.yaml
```

> **OSD 할당 주의사항 (Troubleshooting 참고)**: Rook은 기존 파일시스템 서명(예: ext4)이 남아있는 디스크는 데이터 유실 방지를 위해 포맷하지 않고 건너뜁니다. OSD로 디스크를 사용하려면 배포 전에 각 워커 노드에서 서명을 완벽히 제거해야 합니다.
> ```bash
> # 디스크 서명 제거 예시 (해당 디스크의 데이터를 모두 삭제하므로 주의)
> sudo wipefs -a /dev/nvme0n1
> sudo dd if=/dev/zero of=/dev/nvme0n1 bs=1M count=100 oflag=direct,dsync
> ```

배포 후 Ceph Cluster의 상태가 `HEALTH_OK`인지 확인합니다.
```bash
# Ceph Toolbox Pod에 접속하여 상태 확인
kubectl exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') -n rook-ceph -- ceph status
```

### 2.3 Ceph-Adapter (OpenStack 연동)
Ceph 클러스터 생성이 완료되면, OpenStack 서비스들(Glance, Cinder 등)이 Ceph에 접근할 수 있도록 접속 정보와 인증 키를 `openstack` 네임스페이스로 넘겨주어야 합니다. 이를 위해 OpenStack-Helm의 `ceph-adapter-rook` 차트를 활용합니다.

```bash
# ceph-adapter-rook 배포
helm upgrade --install ceph-adapter-rook openstack-helm/ceph-adapter-rook \
  --namespace openstack \
  --values manifests/overrides/ceph-adapter-rook.yaml
```
이 작업이 끝나면 `openstack` 네임스페이스 안에 `ceph-etc` 등의 Secret/ConfigMap이 생성되며, 이를 통해 OpenStack 서비스들이 Ceph를 스토리지로 사용할 수 있게 됩니다.

## 3. 공식 레퍼런스
* [Rook-Ceph Helm Chart 공식 문서](https://rook.io/docs/rook/latest/Helm-Charts/operator-chart/)
* [OpenStack-Helm Storage Guide](https://docs.openstack.org/openstack-helm/latest/install/storage.html)
