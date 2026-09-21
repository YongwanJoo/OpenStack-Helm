# Horizon Reverse Proxy, TLS, Session 점검

## 주요 증상

- HTTPS 접속 후 HTTP 주소로 다시 redirect됨
- 로그인 직후 다시 로그인 화면으로 돌아감
- CSRF 또는 origin 관련 오류가 발생함
- 내부 주소에서는 되지만 외부 프록시 경로에서는 실패함

## 확인 순서

### 1. OpenStack endpoint

```bash
openstack endpoint list
```

Horizon이 참조하는 Keystone endpoint와 사용자가 접속하는 외부 주소 체계가 일관적인지 확인합니다.

### 2. Proxy 전달 정보

OCI HAProxy와 Envoy Gateway를 차례로 거치는 환경에서는 원래 요청의 host와 HTTPS protocol 정보가 Horizon까지 전달돼야 합니다. 각 계층의 access log에서 동일 요청을 추적합니다.

### 3. TLS와 cookie

- TLS를 어느 계층에서 종료하는지 확인합니다.
- 외부 주소가 HTTPS라면 secure cookie 정책과 충돌하지 않는지 확인합니다.
- 허용 host/origin 범위에 실제 외부 도메인이 포함되는지 확인합니다.
- 프록시 계층마다 redirect를 중복 적용하지 않는지 확인합니다.

실제 도메인, 인증서, Secret, 세션 키는 저장소에 커밋하지 않습니다.

### 4. 애플리케이션 로그

```bash
kubectl logs -n openstack -l application=horizon --tail=200
kubectl get events -n openstack --sort-by=.lastTimestamp
```

## 검증

브라우저에서 외부 HTTPS 주소 접속, Keystone 로그인, 인스턴스 목록 조회, 로그아웃을 순서대로 확인합니다. 최종 환경에서는 이 전체 흐름이 정상 동작했습니다.
