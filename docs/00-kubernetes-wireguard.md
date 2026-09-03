# 0단계: Kubernetes & WireGuard 환경 구성

OpenStack-Helm을 올리기 위한 기반 인프라인 WireGuard VPN과 Kubernetes(kubeadm) 클러스터를 구성하는 과정입니다. 여러 환경(홈 랩, OCI 클라우드)에 흩어진 노드들을 하나의 사설 네트워크로 묶고, 그 위에 Kubernetes 클러스터를 설치합니다.

## 1. 운영체제 및 인프라 변경 이력

* **운영체제 변경**: 초기 Rocky Linux 9 환경에서 진행하려 했으나, 멘토님의 조언에 따라 OpenStack 호환성이 더 뛰어난 **Ubuntu 24.04 LTS**로 전면 교체했습니다.
* **네트워크 대역 설계**:
  * **WireGuard (노드 간 통신)**: `10.0.0.0/24`
  * **Kubernetes Pod Network**: `10.244.0.0/16`
  * **Kubernetes Service Network**: `10.96.0.0/12`

## 2. WireGuard Hub 구성 및 이전

다양한 환경의 노드들을 묶기 위해 공인 IP를 가진 OCI VM 중 하나(`10.0.0.8`)를 **WireGuard Hub (NAT 인스턴스)**로 사용합니다. 인터넷을 통한 VPN 연결이므로 패킷 단편화 방지를 위해 모든 노드의 **WireGuard MTU를 1420**으로 고정했습니다.

### 워커 노드(Spoke) 구성 예시
각 워커 노드(`10.0.0.x`)는 아래와 같은 설정을 사용해 OCI Hub(`168.110.107.75:51820`)를 Endpoint로 바라보게 구성합니다.

```ini
[Interface]
Address = 10.0.0.10/24
PrivateKey = <Private Key>
MTU = 1420

[Peer]
PublicKey = <OCI Hub Public Key>
Endpoint = 168.110.107.75:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```

## 3. Kubernetes (kubeadm) 클러스터 구성

OCI VM 리소스(1vCPU, 1GB RAM)는 Control Plane으로 쓰기에 부족하므로 WireGuard 중계 전용으로 두고, **물리 미니 PC(`10.0.0.7`)를 Control Plane으로 지정**합니다.

### 3.1 노드 공통 사전 설정 (Baseline)
모든 노드(Control Plane, Worker)에서 커널 파라미터와 Swap을 설정합니다.

```bash
# Swap 비활성화
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 커널 모듈 로드
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl 설정 (IPv4 포워딩 및 브릿지 트래픽 허용)
sudo tee /etc/sysctl.d/99-kubernetes-cri.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### 3.2 Container Runtime (containerd) 설치 및 설정
Ubuntu 24.04의 공식 리포지토리에서 `containerd`(v2.2.1)를 설치합니다.

```bash
sudo apt-get update
sudo apt-get install -y containerd

# 기본 설정 파일 생성 및 SystemdCgroup 활성화
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```
> **버전 충돌 주의**: 수동 설치한 `containerd`와 apt 패키지 버전이 혼재할 경우 config 버전 에러가 발생할 수 있습니다. `/usr/bin/containerd` 경로의 버전을 사용하는지 확인해야 합니다.

### 3.3 kubeadm, kubelet, kubectl 설치 (v1.33.13)
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Kubernetes 1.33 리포지토리 추가 (공식 문서 가이드 참조)
# ... 키 링 추가 과정 생략 ...

sudo apt-get update
sudo apt-get install -y kubeadm=1.33.13-1.1 kubelet=1.33.13-1.1 kubectl=1.33.13-1.1
sudo apt-mark hold kubeadm kubelet kubectl
```

### 3.4 Control Plane 초기화 (`kubeadm init`)
API Server가 WireGuard IP(`10.0.0.7`)를 바라보게 하기 위해 `kubeadm-init.yaml`을 생성하여 초기화합니다.

```yaml
# /root/kubeadm-init.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.0.0.7"
  bindPort: 6443
nodeRegistration:
  name: "openstack-jw-01"
  criSocket: "unix:///run/containerd/containerd.sock"
  kubeletExtraArgs:
    - name: "node-ip"
      value: "10.0.0.7"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.33.13"
controlPlaneEndpoint: "10.0.0.7:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

```bash
sudo kubeadm config images pull --config /root/kubeadm-init.yaml
sudo kubeadm init --config /root/kubeadm-init.yaml
```

### 3.5 CNI (Calico) 설치
WireGuard 네트워크(`10.0.0.0/24`) 위에서 Pod 통신을 처리하기 위해 **Calico VXLAN 모드**를 배포합니다. (MTU는 WireGuard보다 작은 1370으로 설정)

먼저 Calico 오퍼레이터를 설치합니다.
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.7/manifests/tigera-operator.yaml
```

이후 커스텀 리소스를 배포합니다.
```yaml
# calico-custom-resources.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  variant: Calico
  calicoNetwork:
    bgp: Disabled
    mtu: 1370
    nodeAddressAutodetectionV4:
      kubernetes: NodeInternalIP
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLAN
      natOutgoing: Enabled
      nodeSelector: all()
```
```bash
kubectl apply -f calico-custom-resources.yaml
```

### 3.6 Worker Node Join
Control Plane에서 토큰을 생성합니다.
```bash
sudo kubeadm token create --print-join-command
```

각 Worker Node에서는 `kubelet`의 `--node-ip` 파라미터가 자신의 WireGuard IP(예: `10.0.0.6`)를 가리키도록 `/etc/default/kubelet`을 수정한 뒤 `kubeadm join`을 실행합니다.

```bash
sudo kubeadm join 10.0.0.7:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```
