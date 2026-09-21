# 7단계: Tenant & External Network

Neutron Tenant 네트워크의 VM을 OCI의 공인 진입점과 연결한 최종 네트워크 구성 기록입니다. 실제 공인 IP, 인증 정보, 호스트별 관리 주소는 공개하지 않습니다.

## 1. 최종 경로

```text
External Client
  -> OCI Public IP
  -> DNAT / HAProxy
  -> GRE over WireGuard
  -> Neutron external network
  -> Floating IP
  -> OpenStack VM
```

VM의 outbound 경로는 반대 방향으로 Neutron Router, GRE over WireGuard, OCI NAT를 통과합니다.

## 2. Tenant 네트워크 검증

최종 환경에서는 Tenant 네트워크와 Subnet을 만든 뒤 서로 다른 Compute 노드에 VM을 배치해 다음 항목을 확인했습니다.

- VM이 DHCP 주소를 정상 발급받는다.
- 같은 Tenant 네트워크의 VM끼리 통신한다.
- 서로 다른 Compute 노드 간 VXLAN 통신이 성공한다.

```bash
openstack network list
openstack subnet list
openstack port list
openstack server list --long
```

## 3. External Network와 OCI 경로

베어메탈 홈 랩과 OCI 사이의 L2 구간을 직접 확장할 수 없으므로, WireGuard 위에 GRE 터널을 구성해 Neutron external network의 경로를 전달했습니다. OCI에서는 forwarding, 정책 라우팅, NAT/DNAT가 함께 필요합니다.

설정 시 다음 값을 환경별 변수로 관리합니다.

```text
<WG_INTERFACE>
<GRE_INTERFACE>
<EXTERNAL_CIDR>
<OPENSTACK_ROUTER_IP>
<OCI_PUBLIC_INTERFACE>
```

방화벽은 전체 허용 대신 실제 경로에 필요한 WireGuard, GRE, forwarding 트래픽만 허용합니다. 영구 설정을 적용하기 전에 임시 규칙으로 왕복 경로를 검증합니다.

## 4. Floating IP와 외부 SSH

```bash
openstack floating ip list
openstack router list
openstack router show <ROUTER_NAME>
```

완료 판정은 다음 결과를 모두 만족하는 경우입니다.

- VM에서 외부 IP로 ping 또는 애플리케이션 요청이 성공한다.
- VM에서 DNS 조회가 성공한다.
- Floating IP를 연결한 VM에 OCI 공인 진입점을 통해 SSH 접속할 수 있다.
- 응답 패킷이 동일한 NAT/GRE/WireGuard 경로로 돌아온다.

## 5. 재부팅 복구

GRE 인터페이스, 정책 라우팅, NAT 규칙, OVS external bridge 연결은 재부팅 후 자동 복구되도록 시스템 서비스로 관리했습니다. 재부팅 검증 시 단순 프로세스 상태뿐 아니라 실제 데이터 경로를 확인합니다.

```bash
sudo systemctl --failed
sudo ip tunnel show
sudo ip rule show
sudo ovs-vsctl show
```

최종적으로 재부팅 후 VM 인터넷, DNS, Floating IP, 외부 SSH, OVS 재연결이 다시 동작하는 것을 확인했습니다.

세부 진단 순서는 [GRE/OCI NAT 트러블슈팅](troubleshooting/04-external-gre-oci-nat.md)을 참고합니다.
