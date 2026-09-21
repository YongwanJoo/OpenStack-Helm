# External Network: GRE over WireGuard와 OCI NAT

## 증상별 확인 순서

외부 네트워크는 여러 계층을 통과하므로 한 번에 NAT 규칙을 바꾸기보다 아래 순서로 경계를 나눠 확인합니다.

```text
VM
 -> Neutron Router
 -> external network
 -> GRE interface
 -> WireGuard
 -> OCI forwarding/NAT
 -> Internet
```

## 1. VM에서 Neutron Router까지

```bash
openstack port list --router <ROUTER_NAME>
openstack router show <ROUTER_NAME>
```

VM의 기본 경로, DHCP, 보안 그룹을 먼저 확인합니다. 이 구간이 실패하면 OCI 설정을 변경해도 해결되지 않습니다.

## 2. GRE와 WireGuard

```bash
sudo ip tunnel show
sudo ip addr show <GRE_INTERFACE>
sudo wg show
sudo ip route show
```

GRE endpoint까지의 underlay 경로가 WireGuard를 통과하는지, 터널 양쪽의 local/remote 값과 MTU가 일치하는지 확인합니다.

## 3. OCI forwarding과 NAT

```bash
sudo sysctl net.ipv4.ip_forward
sudo ip rule show
sudo ip route show table all
```

OCI 측에서는 IP forwarding, source NAT, inbound DNAT, 정책 라우팅, 클라우드 보안 규칙을 함께 확인합니다. 실제 공인 IP와 NAT 원문은 저장소에 남기지 않습니다.

## 4. 왕복 경로 검증

- VM에서 외부 IP와 DNS를 각각 테스트합니다.
- Floating IP를 연결하고 외부 SSH를 테스트합니다.
- 요청과 응답이 같은 GRE/WireGuard 경로를 사용하는지 확인합니다.
- 임시 규칙으로 성공한 뒤 재부팅 가능한 영구 설정으로 전환합니다.

최종 환경에서는 GRE over WireGuard와 OCI NAT/DNAT를 통해 VM 인터넷, DNS, Floating IP, 외부 SSH를 모두 확인했습니다.
