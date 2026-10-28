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
