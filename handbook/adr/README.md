# ADR (아키텍처 결정 기록)

되돌리기 어렵거나 여러 레포에 영향을 주는 결정을 기록한다. 결정을 바꿀 때는 기존 ADR을 고치지 않고 새 ADR을 쓴 뒤, 기존 ADR의 상태를 "ADR XXXX로 대체됨"으로 바꾼다.

## 템플릿

```markdown
# ADR NNNN: 제목

- 상태: 제안 / 채택 / ADR XXXX로 대체됨
- 날짜: YYYY-MM-DD

## 맥락
## 결정
## 결과
## 검토한 대안
```

## 목록

- [ADR 0001: 레포 구조: 제품 2 + 플랫폼 1 + 외부 1 + 테스트 도구 1 + 조직 메타](0001-레포-구조.md)
- [ADR 0002: 백엔드 스택: NestJS (TypeScript)](0002-백엔드-스택.md)
- [ADR 0003: API 계약: 스펙 우선 (OpenAPI + AsyncAPI)](0003-API-계약.md)
- [ADR 0004: 이벤트 브로커: Kafka + 트랜잭셔널 아웃박스, 지연 처리는 DB 스윕](0004-이벤트-브로커.md)
- [ADR 0005: ORM: Drizzle ORM + drizzle-kit](0005-ORM.md)
- [ADR 0006: 검색: OpenSearch + Nori를 처음부터 도입](0006-검색.md)
- [ADR 0007: 서비스 형태: 멀티 판매자 마켓플레이스, 한 계정이 구매자 겸 판매자](0007-서비스-형태.md)
- [ADR 0008: LLM: 제공자 독립 계층](0008-LLM.md)
- [ADR 0009: 결제: PG 추상화 + 토스페이먼츠 테스트 모드 / 모의 PG](0009-결제.md)
- [ADR 0010: 배포: Oracle Cloud Always Free ARM, 무료 구성](0010-배포.md)
- [ADR 0011: 협업 방식: Claude가 아주 작게 구현하고 내가 세밀하게 리뷰한다](0011-협업-방식.md)
- [ADR 0012: 스펙 우선 툴체인 확정: Hey API nestjs 타입 + express-openapi-validator + Problem Details 필터](0012-스펙-우선-툴체인-확정.md)
- [ADR 0013: 스펙 공통 규약: camelCase + RFC 9457 Problem Details + AIP-158 페이지네이션, Redocly 린트로 강제](0013-스펙-공통-규약.md)
- [ADR 0014: 배포: 로컬 PC + Cloudflare Tunnel, pull 방식 배포](0014-배포-로컬-PC와-Cloudflare-Tunnel.md)
- [ADR 0015: 인증: Better Auth, DB 세션, 운영자는 별도 인스턴스](0015-인증-Better-Auth.md)
- [ADR 0016: 이벤트 계약: CloudEvents binary 봉투, 양방향 호환성 검사, Schema Registry 없음](0016-이벤트-계약.md)
- [ADR 0017: API 클라이언트 생성 도구: Hey API, 서버 호출과 오류 정규화는 공용 코드로](0017-API-클라이언트-생성-도구.md)
- [ADR 0018: 웹 앱의 API 주소: 빌드에 박지 않고 실행 시점에 환경 변수로 주입한다](0018-웹-API-주소-실행-시점-주입.md)
- [ADR 0019: SPA 빌드 도구: Vite](0019-SPA-빌드-도구-Vite.md)
- [ADR 0020: 웹 앱의 API 주소: 빌드 시 프레임워크 관례대로 주입한다](0020-웹-API-주소-빌드-시-주입.md)
- [ADR 0021: SPA 라우터: TanStack Router](0021-SPA-라우터-TanStack-Router.md)
- [ADR 0022: Cloudflare Tunnel 구성과 하위 도메인](0022-Cloudflare-Tunnel-구성과-하위-도메인.md)
- [ADR 0023: 비밀값 관리: PC의 권한 제한된 파일이 원본, 비밀번호 관리자에 백업](0023-비밀값-관리.md)
- [ADR 0024: 이미지 빌드와 GHCR 게시: arm64 호스팅 러너, 커밋 SHA 태그, 공개 패키지](0024-이미지-빌드와-GHCR-게시.md)
- [ADR 0025: 배포 에이전트: infra 레포가 배포 상태를 소유하고 PC가 따라간다 (GitOps)](0025-배포-에이전트와-배포-상태.md)
- [ADR 0026: staging 환경: 상한을 줄여 상시 운영하고 기능 확인에만 쓴다](0026-staging-환경-크기와-분리.md)
