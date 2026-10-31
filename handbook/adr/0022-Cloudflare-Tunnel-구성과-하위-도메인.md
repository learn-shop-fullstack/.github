# ADR 0022: Cloudflare Tunnel 구성과 하위 도메인

- 상태: 채택
- 날짜: 2026-10-31

## 맥락

외부 공개는 Cloudflare Tunnel(named tunnel)로만 하고 보유 도메인 `p-stack.dev`의 하위 도메인을 쓰기로 했다(ADR 0014). 하위 도메인 구성, 터널 설정을 어디서 관리할지, 리버스 프록시가 필요한지는 1단계로 미뤘다.

- 공개할 앱은 api, store, seller, admin이고 환경은 production과 staging이다(같은 PC의 별도 compose 프로젝트).
- Cloudflare 무료 인증서(Universal SSL)는 루트와 한 단계 하위 도메인(`api.example.com`)만 덮는다. `api.staging.example.com` 같은 두 단계 이름은 유료 인증서가 필요하다.
- 터널 설정(호스트명 → 서비스)은 대시보드에 두는 방식(remotely-managed, Cloudflare 권장)과 설정 파일로 두는 방식(locally-managed)이 있다.
- infra 레포 규칙: PC에서 손으로 바꾼 운영 설정은 레포에 반영될 때까지 미완료로 본다.

## 결정

- **호스트명**: `p-stack.dev`의 한 단계 하위 도메인에 프로젝트 접두어를 붙인다. 도메인의 다른 이름은 다른 프로젝트에 남긴다.

  | 앱 | production | staging |
  |---|---|---|
  | api | `shop-api.p-stack.dev` | `shop-staging-api.p-stack.dev` |
  | store | `shop-store.p-stack.dev` | `shop-staging-store.p-stack.dev` |
  | seller | `shop-seller.p-stack.dev` | `shop-staging-seller.p-stack.dev` |
  | admin | `shop-admin.p-stack.dev` | `shop-staging-admin.p-stack.dev` |

- **터널**: 환경마다 named tunnel 하나(`shop-sim-production`, `shop-sim-staging`). `cloudflared`는 그 환경의 compose 프로젝트 안에서 돌고 서비스 이름(`api:3000` 등)으로 연결한다.
- **설정 관리**: locally-managed. 호스트명 → 서비스 매핑(ingress)은 shop-sim-infra의 설정 파일로 두고 PR로 리뷰한다. 터널 자격 증명(JSON)만 레포 밖의 비밀 파일이다.
- **DNS**: 호스트명마다 터널을 가리키는 CNAME(proxied)을 둔다. 터널을 만들 때 한 번 등록한다.
- **리버스 프록시**: 두지 않는다. 호스트명 라우팅은 터널 ingress가 하고, TLS는 Cloudflare 엣지에서 끝난다. 터널과 서비스 사이는 compose 내부 네트워크의 HTTP다.

## 결과

- 무료 인증서로 두 환경의 모든 호스트명이 HTTPS다
- 공개 경로(호스트명 → 서비스)가 레포에 있어 바뀐 이력과 리뷰가 남는다
- 서비스가 늘면 ingress 한 줄과 DNS 레코드 하나를 더한다. 프록시 설정은 없다
- 대가와 보완책
  - Cloudflare 권장(remotely-managed)과 다르다 → 대시보드에서 매핑을 고치지 않는다. 설정 파일이 유일한 원본이다
  - 터널 생성·DNS 등록은 계정 인증서(`cert.pem`)가 있는 사람이 손으로 한 번 한다 → 절차를 infra 문서로 남기고, `cert.pem`은 컨테이너와 레포에 두지 않는다
  - 자격 증명 JSON을 잃으면 그 환경의 터널이 뜨지 않는다 → 비밀값 관리 항목에서 보관 위치와 재발급 절차를 정한다
  - 호스트명이 web 빌드의 API 주소(ADR 0020)와 api의 CORS·쿠키 정책의 입력이 된다 → 호스트명을 바꾸면 web을 다시 빌드하고 api 설정을 함께 바꾼다
  - 경로 기반 라우팅, 헤더 조작, 압축 같은 프록시 기능이 필요해지면 → 리버스 프록시를 다시 검토한다

## 검토한 대안

- **remotely-managed(대시보드·토큰)**: Cloudflare 권장이고 컨테이너는 토큰만 받는다. 그러나 매핑이 레포 밖에 있어 리뷰와 이력이 남지 않는다
- **remotely-managed + Terraform**: 터널·DNS·ingress를 모두 코드로 관리한다. 그러나 도구와 상태 파일, API 토큰 관리가 새로 들어온다. 서비스 수가 적은 지금은 이득보다 비용이 크다
- **두 단계 하위 도메인(`api.staging.p-stack.dev`)**: 이름이 깔끔하다. 그러나 무료 인증서가 덮지 않아 유료 인증서가 필요하다
- **접두어 없는 이름(`api.p-stack.dev`)**: 짧다. 그러나 도메인의 최상위 이름을 이 프로젝트가 차지한다
- **두 환경이 터널 하나를 공유**: 터널 수가 줄지만 한 `cloudflared`가 두 compose 프로젝트의 네트워크에 붙어야 해 환경 분리가 흐려진다
- **리버스 프록시(Caddy) 앞단**: 공개 진입점이 하나가 되고 프록시 기능을 쓸 수 있다. 그러나 터널 ingress가 이미 호스트명 라우팅을 하고 TLS도 엣지에서 끝나 지금은 겹친다
