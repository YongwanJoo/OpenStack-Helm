# 4단계: OpenStack-Helm Backend 설치

본격적인 OpenStack 서비스(Nova, Neutron 등)를 띄우기에 앞서, 서비스들이 공통으로 사용하는 기반 인프라(메시지 큐, 데이터베이스, 캐시)를 먼저 배포합니다. 이 서비스들은 OpenStack 컴포넌트 간의 통신과 상태 저장을 책임지는 뼈대 역할을 합니다.

## 1. MariaDB (데이터베이스) 배포

OpenStack의 거의 모든 서비스(Keystone, Nova, Neutron, Cinder 등)는 각각 자신의 상태와 메타데이터를 저장하기 위해 관계형 데이터베이스를 필요로 합니다. OpenStack-Helm에서는 고가용성(HA) 구성이 가능한 MariaDB(Galera Cluster)를 기본 스토리지 엔진으로 사용합니다.

### 설치 명령어
```bash
# 기본 설정 다운로드 (get-values-overrides 활용)
helm osh get-values-overrides -p /path/to/manifests -c mariadb

# 배포 실행
helm upgrade --install mariadb openstack-helm/mariadb \
  --namespace openstack \
  -f manifests/overrides/mariadb.yaml \
  -f manifests/custom/mariadb.yaml
```
배포 후 MariaDB 파드가 Ceph-CSI 기반의 PVC(`general` StorageClass)를 정상적으로 할당받았는지 확인합니다.
```bash
kubectl get pvc -n openstack | grep mariadb
```

## 2. RabbitMQ (메시지 큐) 배포

OpenStack 서비스들은 컴포넌트 간 통신(RPC, Remote Procedure Call)을 위해 AMQP 기반의 메시지 큐 시스템을 광범위하게 사용합니다. 예를 들어, Nova-API가 VM 생성 요청을 받으면 메시지 큐를 통해 Nova-Compute 노드로 작업을 위임합니다.

### 커스텀 설정 (Replica 조정)
우리 클러스터 규모나 리소스 상황에 맞게 Replica 수를 1개로 고정하여 가볍게 운영할 수 있습니다. `manifests/custom/rabbitmq.yaml`을 생성하여 다음 내용을 추가합니다.
```yaml
pod:
  replicas:
    server: 1
```

### 설치 명령어
```bash
# 배포 실행
helm upgrade --install rabbitmq openstack-helm/rabbitmq \
  --namespace openstack \
  -f manifests/overrides/rabbitmq.yaml \
  -f manifests/custom/rabbitmq.yaml
```

## 3. Memcached (캐싱) 배포

OpenStack의 인증 토큰 관리 서비스(Keystone) 및 기타 API 서비스들은 성능 향상을 위해 토큰 데이터나 세션 상태를 임시로 캐싱해야 합니다. 데이터베이스(MariaDB) 부하를 대폭 줄여주며 인메모리 방식이므로 응답 속도가 빠릅니다.

### 설치 명령어
```bash
helm upgrade --install memcached openstack-helm/memcached \
  --namespace openstack \
  -f manifests/overrides/memcached.yaml
```

---

## 4. 파드 기동 대기 및 상태 확인

> **배포 순서 및 대기 (`wait-for-pods`)**
> Backend 서비스들은 매우 중요한 의존성을 가집니다. MariaDB와 RabbitMQ가 100% 정상 기동되기 전에 Keystone이나 Nova를 배포하면 Database 연결 실패 및 Queue 생성 실패로 인해 파드들이 `CrashLoopBackOff` 상태에 빠지게 됩니다. 

모든 백엔드 파드가 `Ready` 상태가 될 때까지 기다리기 위해 OpenStack-Helm 플러그인의 기능을 사용합니다.
```bash
# openstack 네임스페이스 내의 필수 인프라 파드가 모두 뜰 때까지 대기
helm osh wait-for-pods openstack
```

대기가 정상적으로 끝나면, 서비스가 제대로 동작 중인지 확인합니다.
```bash
kubectl get pods -n openstack
```

## 5. 공식 레퍼런스
* [OpenStack-Helm Infrastructure Guide](https://docs.openstack.org/openstack-helm/latest/install/infrastructure.html)
