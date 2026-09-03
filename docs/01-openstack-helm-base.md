# 1단계: OpenStack-Helm 기반 구성 요소 설치

OpenStack-Helm(Helm 3.21.3) 차트 배포와 설정 관리를 위한 초기 구성 단계입니다. 수천 줄에 달하는 OpenStack 차트의 기본 설정값들을 가져오고, 클러스터에 배포할 네임스페이스 및 노드 라벨링을 준비합니다.

## 1. Helm Plugin (`osh`) 설치 및 Repository 설정

OpenStack-Helm은 기본 Helm 명령어만으로는 처리하기 까다로운 방대한 설정 관리 및 배포 과정을 돕기 위해 전용 플러그인을 제공합니다.

### 플러그인 설치 및 리포지토리 추가
```bash
helm plugin install https://opendev.org/openstack/openstack-helm-plugin
helm repo add openstack-helm https://tarballs.opendev.org/openstack/openstack-helm
helm repo update
```
* **`get-values-overrides`**: 가장 핵심적인 플러그인 기능 중 하나입니다. 공식 Repository에서 배포 환경(Ubuntu 등)에 맞는 최적의 `values.yaml` (Overrides)을 자동으로 다운로드하고 조합해 줍니다. 우리는 이 베이스 위에 우리만의 `custom/values.yaml`만 덮어쓰면 되므로 관리가 매우 쉬워집니다.
* **`wait-for-pods`**: 특정 네임스페이스의 모든 Pod가 `Ready` 상태가 될 때까지 기다려주는 기능으로, 배포 스크립트 작성 시 서비스 간(예: RabbitMQ → Nova) 순서를 제어하는 데 필수적입니다.

## 2. Namespace 생성 및 Node Labeling

OpenStack 컴포넌트들이 적절한 노드에 스케줄링(배치)될 수 있도록 미리 정해진 Label과 Taint를 설정합니다.

### 네임스페이스 생성
```bash
kubectl create namespace openstack
```

### 노드 라벨링 (모든 워커 노드 대상)
```bash
# Control Plane 역할 활성화
kubectl label --overwrite nodes --all openstack-control-plane=enabled

# Compute Node (하이퍼바이저) 역할 활성화
kubectl label --overwrite nodes --all openstack-compute-node=enabled

# Open vSwitch 네트워크 데몬 역할 활성화
kubectl label --overwrite nodes --all openvswitch=enabled
```

> **Taint 해제 (필요시)**: 만약 미니 PC 등 자원 제약으로 인해 Control Plane 노드에도 OpenStack 파드를 띄워야 한다면, Control Plane의 Taint를 해제해야 합니다.
> ```bash
> kubectl taint nodes -l 'node-role.kubernetes.io/control-plane' node-role.kubernetes.io/control-plane-
> ```

## 3. GitOps 기반 디렉토리 구조 (Repository Architecture)

인프라의 멱등성과 재현성을 보장하기 위해 Git을 통해 설정값 및 매니페스트를 선언적으로 관리합니다.

```text
OpenStack-Helm/
├── versions.env          # 클러스터 컴포넌트(Helm chart) 버전 정의 
├── env.sh                # 공통 환경 변수 및 플래그 정의
├── manifests/            # 인프라 설정 및 배포 매니페스트 모음
│   ├── overrides/        # 'get-values-overrides'로 다운로드한 공식 기본 설정
│   ├── custom/           # 사용자 정의 설정 (Replica 조정, PVC 설정 등)
│   └── infrastructure/   # 인프라 기반 서비스들 (MetalLB, Envoy, Ceph 등)
└── scripts/              # 배포 자동화 스크립트
```

### 배포 원리
1. `env.sh`와 `versions.env`를 소싱하여 환경 변수 적용
2. 플러그인을 사용해 `manifests/overrides/`에 공식 베이스 설정 생성
3. 배포 시 `helm install ... -f manifests/overrides/기본.yaml -f manifests/custom/내설정.yaml` 형태로 오버라이딩

이 구조를 통해 인프라 코드를 깨끗하게 유지하며, 향후 설정 변경 시 Git History를 통해 쉽게 추적할 수 있습니다.
