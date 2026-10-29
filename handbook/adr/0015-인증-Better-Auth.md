# ADR 0015: 인증: Better Auth, DB 세션, 운영자는 별도 인스턴스

- 상태: 채택
- 날짜: 2026-10-28
- 정정: 2026-10-29 인증 경로 표기에서 `/api` 접두어를 뺐다. API를 하위 도메인으로 공개해 경로 접두어를 두지 않기로 했기 때문이다(shop-sim-api#26). 결정 내용은 같다

## 맥락

2단계에 소셜 로그인(Google·Kakao·Naver), 세션 vs JWT 결정, 셀러센터 SSO, 일반 계정과 분리된 운영자 계정이 있다. 인증을 직접 구현하면 세션·CSRF·계정 연결에서 보안 실수의 여지가 크다. Better Auth를 POC(shop-sim-api#17)로 검증했다.

## 결정

- **라이브러리**: `better-auth`(정확한 버전 고정, POC는 1.7.14). Nest에는 Express 핸들러(`toNodeHandler`)로 직접 연결하고 가드로 세션을 조회한다. 커뮤니티 Nest 모듈은 쓰지 않는다.
- **세션**: DB 세션 + 서명된 HttpOnly 쿠키(`SameSite=Lax`, HTTPS에서 `Secure`). JWT는 쓰지 않는다.
- **SSO**: 일반 계정 세션 쿠키를 상위 도메인에 걸어 스토어·셀러센터가 공유한다. 신뢰 출처 목록 밖의 Origin은 거부된다. staging은 별도 쿠키 도메인을 쓴다.
- **운영자 계정**: 별도 Better Auth 인스턴스(별도 경로·`admin_*` 테이블·쿠키 이름·비밀값). 운영자 쿠키는 운영자 호스트에만 건다. admin 플러그인의 `role` 방식은 쓰지 않는다.
- **스키마**: core 테이블을 Drizzle로 직접 정의해 ERD 규칙(복수 테이블, `uuidv7()`, `timestamptz`, 제약 이름)을 따른다. 이메일 가입도 `accounts`의 한 행(`credential`)이다.
- **스펙 우선의 예외**: `/auth/*`, `/admin/auth/*`는 Better Auth가 계약을 소유한다. OpenAPI 스펙에 넣지 않고 런타임 검증에서 제외하며, 오류 형식은 Better Auth의 `{ message, code }`다. web은 이 경로에 Better Auth 클라이언트를 쓰고 서버와 같은 버전으로 고정한다.

## 결과

- 소셜 제공자 3종이 내장돼 있고, 로그아웃·강제 만료가 세션 행 삭제로 즉시 반영된다
- 구매자 계정이 운영자가 될 경로가 구조적으로 없다
- 대가와 보완책
  - 인증 엔드포인트는 우리 스펙·Problem Details 밖이다 → 클라이언트는 인증 경로만 `code`로 분기, 버전 고정
  - 업그레이드·플러그인 추가 때 스키마를 직접 맞춰야 한다 → `@better-auth/cli generate`로 기준 스키마를 만들어 비교
  - 인스턴스 두 개의 설정·가드를 유지한다 → 테이블은 접두사를 받는 함수 하나로 만든다
  - 요청마다 세션을 DB에서 조회한다 → 부하가 문제 되는 시점(7단계)에 캐시 검토
  - 운영자 가입은 막고 첫 운영자는 시드로 만든다(2단계에서 서버 측 생성 방법 확인)
- Redocly `security-defined`는 인증을 구현하는 2단계에서 켠다(ADR 0013의 "4단계"는 오기)

## 검토한 대안

- Passport(`@nestjs/passport`) + 직접 세션: Nest 공식이고 흐름을 다 배우지만, 세션·CSRF·계정 연결을 직접 구현해야 한다
- Auth.js: Next.js 중심이라 Nest 백엔드가 인증을 소유하는 구조와 맞지 않는다
- JWT: 서버 상태가 없지만 로그아웃·강제 만료가 만료 시각까지 반영되지 않는다
- admin 플러그인(같은 `users` + `role`): 회원 정지·대리 로그인이 내장이지만, 역할 문자열 하나로 운영자 권한이 생기고 운영자 세션이 하위 도메인 공유 쿠키에 실린다
- `@thallesp/nestjs-better-auth`: 가드·데코레이터가 있지만 관리자가 한 명이고 GraphQL·WebSocket까지 peer 의존성이다
