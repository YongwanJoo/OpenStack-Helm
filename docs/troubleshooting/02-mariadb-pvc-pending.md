# MariaDB PVC Pending 및 Ceph-CSI 장애 트러블슈팅

OpenStack 백엔드인 MariaDB를 배포할 때 경험했던 Ceph RBD 볼륨 프로비저닝 장애와 해결 과정을 정리한 문서입니다.

---

## 증상
OpenStack MariaDB를 배포한 직후, 파드(`mariadb-server-0`)가 계속해서 `Pending` 상태에 머물러 있었습니다. 확인 결과 Ceph RBD 스토리지를 요청한 PVC(`mysql-data-mariadb-server-0`)가 오랜 시간 `Pending` 상태로 볼륨을 할당받지 못하고 있었습니다.

## 진단 과정

1. **StorageClass 확인**
   PVC는 `general`이라는 이름의 StorageClass를 정상적으로 요청하고 있었으며, 해당 StorageClass는 `rook-ceph.rbd.csi.ceph.com` (CSI 프로비저너)를 바라보고 있었습니다. 설정 자체에는 문제가 없었습니다.
2. **CSI Plugin 상태 확인**
   Rook-Ceph 네임스페이스(`rook-ceph`)의 CSI 파드들을 조회해보니 `ceph-csi-controller-manager`는 떠 있었으나, 정작 RBD 볼륨을 만들어주는 핵심 데몬인 **`rook-ceph.rbd.csi.ceph.com-ctrlplugin`이 0/1 상태**였고, **Node Plugin(DaemonSet)은 0개**가 동작 중이었습니다.
3. **이벤트(Event) 로그로 원인 확인**
   해당 CSI Plugin 파드 생성 실패 이벤트를 조회해 보니 다음 에러가 발생하고 있었습니다.
   ```text
   pods "rook-ceph.rbd.csi.ceph.com-ctrlplugin-..." is forbidden: error looking up service account rook-ceph/rbd-ctrlplugin-sa: serviceaccount "rbd-ctrlplugin-sa" not found
   ```
   **직접 원인**: CSI 컨트롤러/노드 플러그인 파드가 실행될 때 필요한 권한(ServiceAccount 및 RBAC)이 클러스터에 아예 존재하지 않아 Kubernetes가 파드 생성을 거부하고 있었습니다.
4. **Helm Release 점검**
   이 ServiceAccount들을 만들어 주어야 할 `ceph-csi-drivers` Helm Release가 설치되어 있지 않은 것을 확인했습니다. 이전 과정에서 CSI Driver CR(Custom Resource) 매니페스트만 `kubectl apply`로 단순 적용하여, 껍데기 설정(Driver CR)은 존재하지만 구동에 필수적인 권한 객체들(RBAC/ServiceAccount)이 싹 빠져버린 것이 문제였습니다.

## 해결 방법

1. **잘못 생성된 껍데기 Driver CR 제거**
   기존에 잘못 적용된 CR만 수동으로 삭제했습니다. (단, 실제 스토리지 클래스나 다른 중요한 데이터는 건드리지 않음)
   ```bash
   kubectl -n rook-ceph delete driver.csi.ceph.io rook-ceph.rbd.csi.ceph.com
   ```
2. **공식 Ceph-CSI Drivers 차트로 재설치**
   Rook 버전(v1.20.6)과 호환되는 공식 Ceph-CSI Operator(1.0.4) 차트를 Helm을 통해 정식으로 설치했습니다. 이 과정에서 ServiceAccount와 RBAC 권한이 모두 정상적으로 생성되었습니다.
   ```bash
   helm upgrade --install ceph-csi-drivers ceph-csi-operator/ceph-csi-drivers \
     --version 1.0.4 \
     --namespace rook-ceph \
     -f <https://raw.githubusercontent.com/rook/rook/v1.20.6/deploy/charts/ceph-csi-drivers/values.yaml>
   ```

조치 후 CSI 컨트롤러와 노드 플러그인이 즉각적으로 기동(`Running`)되었고, Pending 상태이던 MariaDB PVC도 RBD 볼륨을 프로비저닝 받아 `Bound` 상태로 변경되며 MariaDB 파드가 정상적으로 배포되었습니다.

---
**주요 교훈 (Lessons Learned)**
> 인프라 컴포넌트 배포 시, 단순한 `kubectl apply`가 아니라 컴포넌트의 라이프사이클과 권한(RBAC)을 모두 책임져 주는 **Helm Chart 단위의 배포(Release)**를 철저히 지켜야 합니다. CRD(Custom Resource) 선언과 실제 구동 데몬의 권한 셋은 구분되어 작동한다는 점을 유의해야 합니다.

---

## 공식 참고 자료

이 문제 해결의 기준이 된 문헌 및 참고 자료입니다. ServiceAccount/RBAC 누락이 CSI Plugin 파드 생성을 막는 현상은 [Rook issue #17644](https://github.com/rook/rook/issues/17644)와도 일치합니다.

* [Ceph-CSI Drivers Helm chart 설치 문서](https://github.com/ceph/ceph-csi-operator/blob/main/docs/helm-charts/drivers-chart.md)
* [Rook v1.20.6 Chart 의존성 정의](https://github.com/rook/rook/blob/v1.20.6/deploy/charts/rook-ceph/Chart.yaml)
* [Rook v1.20.6용 CSI Driver values 파일](https://raw.githubusercontent.com/rook/rook/v1.20.6/deploy/charts/ceph-csi-drivers/values.yaml)
* [Rook RBD Block Storage 구성](https://rook.io/docs/rook/latest-release/Storage-Configuration/Block-Storage-RBD/block-storage/)
