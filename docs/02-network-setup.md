# 2단계: 네트워크 설정 (MetalLB & Envoy Gateway)

OpenStack-Helm에서 외부와의 통신(External Access)을 담당하는 두 핵심 컴포넌트인 **MetalLB**와 **Envoy Gateway**를 배포하고 구성하는 과정입니다.

## 1. MetalLB 배포 및 설정

Kubernetes 클러스터 내부에 띄워진 서비스(Pod)들은 기본적으로 클러스터 내부 IP(ClusterIP)만 가집니다. 클라우드 환경(AWS, GCP 등)에서는 LoadBalancer 타입의 서비스를 만들면 클라우드 제공자가 자동으로 외부 IP를 할당해 주지만, **우리가 구축하는 베어메탈(On-Premise) 환경에서는 이 역할을 해줄 주체가 없습니다.**

이 빈자리를 채워주는 것이 **MetalLB**입니다. 사전에 우리가 지정한 외부 IP 대역(IPAddressPool) 중 하나를 LoadBalancer 서비스에 할당하고, ARP/BGP 프로토콜을 이용해 외부 라우터에 "이 IP는 내가 갖고 있다!"라고 방송(Advertise)하여 트래픽을 끌어옵니다.

### 1.1 설치 명령어
공식 Helm 리포지토리를 추가하고 MetalLB를 배포합니다.
```bash
source scripts/env.sh

helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm install metallb metallb/metallb \
  -n metallb-system \
  --create-namespace \
  --version "$METALLB_VERSION"
```

설치 후 `metallb-system` 네임스페이스의 controller와 speaker 파드가 정상적으로 Running 상태가 되는지 확인합니다.
```bash
kubectl get pods -n metallb-system
```

### 1.2 IPAddressPool 및 L2Advertisement 적용
MetalLB가 IP를 할당하려면 어느 대역의 IP를 사용할지(Pool), 그리고 어떤 방식으로 외부에 알릴지(L2)를 설정해야 합니다. (아래는 홈 랩 네트워크 대역의 예시입니다)

```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: openstack-pool
  namespace: metallb-system
spec:
  addresses:
  # 클러스터 외부에 할당 가능한 사설/공인 IP 대역 입력
  - 172.30.1.200-172.30.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: openstack-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - openstack-pool
```
```bash
kubectl apply -f metallb-config.yaml
```

## 2. Envoy Gateway 배포 및 설정

최근 Kubernetes는 기존의 Ingress API의 한계를 극복하기 위해 더 강력하고 유연한 라우팅을 지원하는 **Gateway API**를 표준으로 밀고 있습니다. **Envoy Gateway**는 이 Gateway API를 구현한 구현체 중 하나로, 실질적인 트래픽 프록시 역할을 수행하는 Envoy Proxy를 관리합니다.

OpenStack은 Identity(Keystone), Compute(Nova), Network(Neutron) 등 수많은 API 엔드포인트를 가집니다. 이 수많은 엔드포인트를 하나의 외부 진입점(Gateway)에서 통합 관리하고 라우팅하기 위해 Envoy Gateway를 사용합니다.

### 2.1 설치 명령어
OCI 레지스트리에서 직접 Helm 차트를 가져와 배포합니다.
```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.0 \
  -n envoy-gateway-system \
  --create-namespace
```

### 2.2 Gateway API 리소스 적용 예시
Envoy Gateway가 동작하려면 Kubernetes 표준 API인 `GatewayClass`와 `Gateway` 리소스가 선언되어야 합니다.

```yaml
# envoy-gateway-config.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openstack-gateway
  namespace: openstack
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
```
```bash
# Gateway API 표준 CRD 설치 (클러스터에 없는 경우)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# 설정 적용
kubectl apply -f envoy-gateway-config.yaml
```
이후 `kubectl get gateway -n openstack` 명령어를 실행하면, **MetalLB가 할당해 준 외부 IP**가 Gateway에 부여된 것을 확인할 수 있습니다.

## 3. 동작 흐름 요약

두 컴포넌트는 서로 협력하여 외부 트래픽을 OpenStack으로 전달합니다.

1. **MetalLB**가 실제 통신 가능한 `외부 IP(External IP)`를 풀에서 할당합니다.
2. **Envoy Gateway**가 이 외부 IP를 물고 있는 `LoadBalancer` 타입의 문(Gateway)을 엽니다.
3. 사용자가 외부 IP로 요청을 보냅니다. (예: `http://<외부IP>/identity`)
4. Envoy Proxy(HTTPRoute 리소스 기반)가 요청 URL(`/identity`)을 분석하고, 내부의 OpenStack `Keystone` 서비스로 라우팅(전달)합니다.

**요약하자면, MetalLB는 "물리적인 주소(IP)를 만들어주는 우체국"이고, Envoy Gateway는 "그 주소로 들어온 트래픽을 각 서비스로 정확히 전달해 주는 안내데스크" 역할을 합니다.**
