# 8단계: Horizon

OpenStack API와 외부 네트워크 검증 이후 Horizon 대시보드를 배포하고 HTTPS 경로를 확인한 기록입니다.

## 1. 배포 전 확인

Horizon은 Keystone 인증과 여러 OpenStack API endpoint에 의존합니다. 먼저 다음 항목을 확인합니다.

```bash
openstack endpoint list
openstack service list
kubectl get pods -n openstack
```

Horizon의 release별 설정은 `manifests/overrides/horizon/2026.1-ubuntu_noble.yaml`에서 관리합니다. 실제 도메인, TLS 키, 비밀번호는 별도 Secret으로 관리하고 저장소에 커밋하지 않습니다.

## 2. 접근 경로

```text
Browser -> HTTPS endpoint -> OCI HAProxy -> WireGuard -> Envoy Gateway -> Horizon
```

이 구성에서 프록시가 전달하는 host와 protocol 정보가 Horizon의 응답 및 세션 cookie와 일치해야 합니다. TLS 종료 위치가 바뀌면 redirect loop, 로그인 직후 세션 만료, CSRF 오류가 발생할 수 있습니다.

## 3. 상태 확인

```bash
helm status horizon -n openstack
kubectl get pods -n openstack -l application=horizon
kubectl get svc,httproute,gateway -n openstack
```

문제가 발생하면 브라우저 오류만 보지 말고 Horizon Pod, Envoy Gateway, OCI HAProxy 순서로 요청이 전달되는지 확인합니다.

```bash
kubectl logs -n openstack -l application=horizon --tail=200
kubectl get events -n openstack --sort-by=.lastTimestamp
```

## 4. 완료 검증

최종 환경에서 다음 항목을 확인했습니다.

- 외부 HTTPS 주소로 Horizon 페이지에 접근한다.
- Keystone 계정으로 로그인한다.
- 프로젝트의 인스턴스 목록을 조회한다.
- 로그아웃 후 세션이 정상 종료된다.

프록시와 cookie 관련 점검 항목은 [Horizon reverse proxy 트러블슈팅](troubleshooting/05-horizon-reverse-proxy-cookie.md)을 참고합니다.
