# 아키텍처

> 원본은 `.github/handbook/architecture.md`. 결정의 이유는 `adr/`, 작업 순서는 `ROADMAP.md`.

## 시스템 구성

```
                 ┌──────────────── shop-sim-web ────────────────┐
  구매자 ──────▶ │ store (Next.js)  seller (SPA)  admin (SPA)    │
                 └───────────────┬───────────────────────────────┘
                                 │ HTTP (OpenAPI) · SSE · WebSocket
  shop-sim-loadtest ─────────────┤ (봇도 같은 공개 API만 사용)
                                 ▼
                 ┌──────────────── shop-sim-api ────────────────┐
                 │ api (모듈러 모놀리스)      worker              │
                 │   └ outbox ──▶ Kafka ──▶ 컨슈머 그룹들        │
                 │ Postgres · Redis · Kafka · OpenSearch · R2    │
                 └───────┬───────────────────────────┬──────────┘
                         │ HTTP (벤더 OpenAPI)        │ 웹훅
                         ▼                           │
                 ┌──────────── shop-sim-vendor ───────┴──────────┐
                 │ mock-pg (PG사)         logistics-sim (택배사) │
                 └──────────────────────────────────────────────┘
                 토스페이먼츠 (테스트 모드) ◀── api 결제 어댑터

  shop-sim-infra: 위 전부의 compose·배포·관측성·CI 재사용 워크플로
```

## 레포 구성

| 레포 | 성격 | 내용 |
|---|---|---|
| `.github` | 조직 메타 | 핸드북(이 문서, 로드맵, ADR, 작업 방식), 조직 공통 PR·이슈 템플릿 |
| `shop-sim-api` | 제품 (백엔드) | `api`, `worker`, OpenAPI·AsyncAPI 스펙, DB 마이그레이션, ABO 시드 |
| `shop-sim-web` | 제품 (프론트) | `store`, `seller`, `admin`, 공용 패키지(api-client, ui, grid, realtime, tiptap-nodes) |
| `shop-sim-infra` | 플랫폼 | compose(역할별), Caddy, 배포·백업 스크립트, 관측성 설정, CI 재사용 워크플로, 이후 k3s·GitOps |
| `shop-sim-vendor` | 외부 회사 흉내 | `mock-pg`(PG사), `logistics-sim`(택배사). 우리 코드와 완전히 분리 |
| `shop-sim-loadtest` | 테스트 도구 | 트래픽 시뮬레이터(구매자·판매자 봇, 시나리오, 과거 데이터 생성), 컨트롤 패널, k6 |

로컬 작업 공간 — 모든 레포를 한 폴더 아래에 클론한다. 여러 레포에 걸친 작업은 상위 폴더에서 Claude Code를 연다.

```
learn-fullstack/        (git 레포 아님, 작업 공간)
  .github/  shop-sim-api/  shop-sim-web/  shop-sim-infra/  shop-sim-vendor/  shop-sim-loadtest/
```

## 계약 (스펙 우선)

모든 레포 간 연결은 **버전이 붙은 스펙**으로만 한다. 상대 레포의 코드를 import하거나 DB를 직접 읽지 않는다.

| 계약 | 소유 레포 | 형식 | 소비자 |
|---|---|---|---|
| 커머스 HTTP API | shop-sim-api `openapi/` | OpenAPI 3.x | shop-sim-web, shop-sim-loadtest |
| 커머스 이벤트 (Kafka) | shop-sim-api `asyncapi/` | AsyncAPI | api 내부 컨슈머(검색 색인, 알림, 분석 등) |
| 모의 PG API | shop-sim-vendor | OpenAPI (토스페이먼츠와 같은 형태) | shop-sim-api |
| 물류 API + 웹훅 | shop-sim-vendor | OpenAPI | shop-sim-api |

운영 규칙

- 스펙은 **코드에서 생성하지 않고**, YAML 자체를 작성하고 리뷰한다.
- 모든 API 변경의 흐름: **스펙 PR(설계 리뷰) → 머지 → 서버 인터페이스 생성 → 구현 / 동시에 소비자는 목 서버로 개발 → 실서버 연결**
- 스펙은 main 머지 시 **semver 패키지**(GitHub Packages)로 배포한다. 소비 레포는 버전을 고정하고, 업그레이드는 별도 PR로 올린다.
- 하위 호환이 깨지는 변경은 메이저 버전을 올리고, "신규 필드/엔드포인트 추가 → 소비자 전환 → 구 필드 제거" 순서로 진행한다.
- 이벤트는 CloudEvents 1.0 binary 모드로 싣고, 옛 메시지 재처리까지 고려해 더 엄격하다: 선택 필드·enum 값 추가만 호환, 필수 필드 추가·필드 삭제·타입 변경(넓힘 포함)은 같은 채널에 새 메이저 메시지(V2)를 병행 발행한다. CI에서 main과의 호환성을 양방향으로 검사한다(ADR 0016, `shop-sim-api/asyncapi/CONVENTIONS.md`).

## 배포 환경 (무료 구성)

**목표 비용: 월 0원** (ADR 0014, 보유 도메인 사용)

| 용도 | 서비스 | 비고 |
|---|---|---|
| 서버 | **로컬 PC**(Apple Silicon, arm64) + docker compose | staging·production은 같은 PC의 별도 compose 프로젝트 |
| 외부 공개 | Cloudflare Tunnel(named tunnel) | 공유기 포트를 열지 않음. SSE는 `Content-Type: text/event-stream`이어야 스트리밍됨 |
| 이미지 스토리지 | Cloudflare R2 무료 구간 | 이그레스 비용 없음 |
| 컨테이너 레지스트리 | GitHub Container Registry | |
| CI | GitHub Actions | arm64 이미지 빌드 → GHCR 푸시까지. PC가 새 이미지를 받아 교체(pull 방식, self-hosted runner 없음) |
| 에러 추적 | Sentry 무료 플랜 | 자체 호스팅하지 않음 |
| 결제 | 토스페이먼츠 테스트 모드 + 모의 PG | 테스트 키는 실제 결제가 일어나지 않음 |
| 부하 생성(shop-sim-loadtest) | 로컬 PC(운영과 같은 기계) | CPU·메모리를 나눠 쓰므로 측정은 그 조건으로 해석 |
| 도메인 | `p-stack.dev`(Cloudflare DNS) | 하위 도메인 구성은 1단계 |

**12GB 메모리 예산 (목표치, 운영 compose의 메모리 상한 합계. 1단계에서 실측 후 조정)**

| 구성 | 예산 |
|---|---|
| Postgres | 1GB |
| Redis | 0.2GB |
| Kafka | 1GB (힙 512MB) |
| OpenSearch | 2GB (힙 768MB. 번들 플러그인 포함 대기 1.39GiB 실측, shop-sim-infra#4) |
| Node 앱 4개 + Next.js | 2GB |
| 관측성 (Prometheus, Loki, OTel Collector) | 1.5GB |
| 프록시·터널, 여유 | 나머지 약 4.3GB |

Kafka UI, OpenSearch Dashboards는 상시로 띄우지 않고 필요할 때만 실행한다.

**단계별 토폴로지**

- 1~12단계: 로컬 PC 1대, 역할별 compose 파일
- 13단계: **관측성을 별도 compose 프로젝트로 분리**하고 메모리 상한을 따로 건다
- 14단계: 앱 / 데이터 / 관측성으로 나누고 로컬 k3d로 k3s 클러스터 실습

**로컬 PC 구성의 위험과 대비**

- PC가 꺼지거나 잠들면 서비스와 시뮬레이터가 멈춘다 → 절전 해제 등 상시 가동 설정, 멈춤은 장애 실습으로 포스트모템에 남긴다
- public 레포라 PC에 self-hosted runner를 두면 외부 PR이 PC에서 코드를 실행할 수 있다 → runner를 두지 않고 pull 방식으로 배포
- 운영과 개발이 같은 기계다 → 운영 compose에 메모리 상한, 개발용 컨테이너는 운영과 다른 compose 프로젝트로
- 모든 데이터는 재생성 가능하게: 시드 스크립트 + 시뮬레이터 + Postgres 백업(R2)으로 PC를 잃어도 하루 안에 복구

## 도메인 모델 (초안)

엔티티 목록의 원본. 테이블·관계·컬럼은 shop-sim-api `db/ERD.md`.

**회원·판매자**
- **User / Session / Account**: 일반 계정(모든 계정은 구매자로 행동 가능)과 세션, 로그인 수단(비밀번호·Google·Kakao·Naver 각각 Account 1행). Better Auth가 관리(ADR 0015)
- **AdminUser / AdminSession / AdminAccount**: 플랫폼 운영자 계정. 일반 계정과 분리된 별도 인증 인스턴스(ADR 0015)
- **Seller / SellerApplication**: User와 1:1로 연결되는 판매자 프로필(스토어 정보), 입점 신청·심사 상태
- **ShippingPolicy**: 판매자별 기본 배송비, 무료배송 기준, 출고지
- **Address**: 구매자 배송지

**카탈로그·재고**
- **Category**: 트리 구조
- **Product**: 판매자, 이름, 부제, 카테고리, 타입(FINISHED / MADE_TO_ORDER), 가격, 할인율, 상세 본문(Tiptap JSON), 리드타임, 상태(draft/published/scheduled/suspended), version
- **ProductImage**: 원본 키, 리사이즈 상태, 정렬 순서(0번이 대표)
- **ProductOption / OptionValue / Sku / SkuOptionValue**: 선택형 옵션, SKU(옵션 값 조합, 옵션 추가금). 옵션 없는 상품도 기본 SKU 1개. 재고·가격·주문은 항상 SKU를 가리킨다
- **CustomOptionField**: 자유 입력형 제작 옵션 정의
- **ProductRevision**: 상세 본문 버전 이력
- **RawMaterial**: 원재료(판매자 소유), 단위, 리드타임
- **BillOfMaterial**: 제작상품 ↔ 원재료 소요량
- **StockLot**: 완제품·원재료 로트, 수량, 입고일, 유효기한
- **StockLedgerEntry**: 재고 변동 원장(입고, 예약, 확정 차감, 해제, 폐기, 조정)
- **StockReservation**: 주문별 재고 예약 (reserved / confirmed / released)

**주문·결제·배송**
- **Cart / CartItem**: 회원 전용(비회원 장바구니 없음)
- **Checkout**: 결제 단위(주문서), 결제 기한
- **Order**: 판매자별 주문(배송 단위), 상태, 배송비
- **OrderItem**: 가격·할인·제작 옵션 입력값 스냅샷
- **Payment / Refund**: PG 거래 ID, 상태, idempotency key
- **Coupon / CouponIssue / CouponRedemption**: 발행 주체(플랫폼/판매자), 발급·사용 이력. 사용은 Checkout에 기록하고 할인액은 OrderItem별 스냅샷
- **OrderIssue / OrderIssueItem**: 취소·반품 요청 (요청 → 승인/거절 → 완료)
- **Shipment**: 판매자별 주문 단위 택배사, 운송장 번호, 배송 상태, 출고 지연 사유, 마지막 동기화 시각

**이벤트·실시간·소통**
- **OutboxMessage / ProcessedMessage**: 아웃박스 이벤트, 멱등 컨슈머 처리 기록
- **Notification / NotificationReceiver**: 알림 본문, 수신자별 읽음 상태
- **ChatRoom / ChatMessage**: 구매자·판매자 쌍(유니크), 비정규화 필드 / 발신자, 유형(텍스트·주문 카드), 주문 카드의 주문, 읽음 시각
- **Notice / Faq / Inquiry / InquiryMessage / Report**: 고객센터

**리뷰·분석·정산**
- **Review / ReviewImage**
- **ReviewSummary / ReviewSummaryRun**: 상품별 긍정·부정 요약, 마지막 반영 리뷰, 배치 실행 이력(처리·건너뜀·실패, 토큰 사용량)
- **DailySales**: SKU별 일 판매량(상품 단위는 합산)
- **DemandForecast / ReorderRecommendation**: 7일 예측값, 사용 모델·오차 / 재주문점, 추천 수량
- **CommissionPolicy / Settlement / SettlementItem**: 수수료율, 판매자별 월 정산, 정산 대상 주문

**운영**
- **AuditLog**: 어드민·판매자 주요 행위 기록
- **BatchRun**: 배치 공통 실행 이력
- **User.isTestAccount**: 시뮬레이터 봇 계정 표시 (통계·정산에서 구분)

**다른 레포가 소유하는 데이터** (커머스 DB에 두지 않는다)
- shop-sim-vendor: **Tracking / TrackingLog**(운송장·상태 이력), 모의 PG 거래
- shop-sim-loadtest: **SimulationScenario / SimulationRun**

## 결제 구성

```
도메인(주문·결제 서비스) → PaymentGateway 포트
                              ├─ TossPaymentsAdapter  → 토스페이먼츠 (테스트 키)   … 사람이 브라우저로 결제
                              └─ MockPgAdapter        → shop-sim-vendor 모의 PG     … 시뮬레이터 봇, 장애 주입
```

- 국내 커머스는 PG사를 여러 곳 붙이는 경우가 많아서, PG 추상화는 실제 회사 구조와도 같다.
- 모의 PG는 토스페이먼츠와 **같은 요청·응답 형태와 오류 코드**를 쓴다 → 어댑터 차이가 최소화된다.
- 토스 테스트 모드만으로는 봇이 결제할 수 없다(결제 위젯은 브라우저에서 결제창을 거쳐야 승인에 필요한 값이 나온다). 대량 트래픽과 장애 주입도 남의 테스트 서버로는 할 수 없다.
- 라우팅: 봇 계정(`isTestAccount`)은 모의 PG, 사람은 토스. 설정으로 강제 전환 가능.
- 공통 보안: 승인 전에 서버에 저장한 주문 금액과 결제 요청 금액을 비교해 위변조를 막는다. 웹훅은 서명·출처를 검증하고 멱등 처리한다.

## 주문 상태

판매자별 주문(배송 단위) 기준.

`PAYMENT_PENDING → PAID(신규 주문, 주문 확인 대기) → PREPARING(상품 준비중) → READY_FOR_SHIPMENT(배송 준비 완료) → IN_DELIVERY(배송중) → DELIVERED(배송 완료) → PURCHASE_CONFIRMED(구매 확정)`

- `PREPARING ↔ SHIPMENT_DELAYED(출고 지연중)`: 판매자가 사유·예정일 입력 → 구매자 알림
- `PAYMENT_PENDING → PAYMENT_TIMEOUT` (기한 스윕)
- `PAID / PREPARING / SHIPMENT_DELAYED → CANCELLED` (배송 시작 전 취소)
- `DELIVERED → REFUND_REQUESTED → REFUNDED` (반품, OrderIssue 승인 필요)
- `PURCHASE_CONFIRMED`: 배송 완료 후 N일 경과 또는 구매자 확정 → 정산 대상
- 상태 전이는 한 곳(상태 머신)에서만 허용하고, 허용되지 않은 전이는 409로 거절

## 이미지 데이터 전략

- 메인: **Amazon Berkeley Objects (ABO)** — 상품 메타데이터(다국어 제목, 브랜드, 유형, 색상, 소재, 치수 등) + 카탈로그 이미지. AWS 계정 없이 `s3://amazon-berkeley-objects/`에서 접근 가능.
  - 라이선스 표기가 출처마다 다름 (AWS 레지스트리·논문: CC BY-NC 4.0 / 데이터셋 사이트: CC BY 4.0). 비상업 학습용으로 사용하고, 스토어 푸터에 출처 표기.
  - 전체 약 300GB → 카테고리 몇 개, 상품 1~2천 개만 선별.
- 보조: Unsplash/Pexels API (상세페이지 연출 컷·배너, 각 API 출처 표기 규칙 확인), AI 생성(배너·로고), placeholder(1단계).
- 한국식 긴 상세페이지 이미지는 없으므로 ABO 사진 + 스톡 이미지를 Tiptap으로 조합해 직접 구성 → 에디터 테스트 케이스로 활용.

## AG Grid 전략

- Community(MIT): 정렬, 필터, 셀 편집, 페이지네이션, Infinite Row Model, 커스텀 컴포넌트, CSV 내보내기 → 배포본은 여기까지만.
- Enterprise(상용 라이선스): SSRM, 행 그룹핑·집계, 피벗, Excel 내보내기, Master/Detail, 통합 차트 → 라이선스 키 없이 로컬 테스트 가능(콘솔 경고·워터마크 표시). 로컬 실험 브랜치에서만.
- 활용처: 셀러센터(상품 목록, SKU 매트릭스, 재고·로트, 주문/배송, 정산), 어드민(입점 심사, 회원·판매자, 전체 주문, DLT, 정산 현황, 실시간 주문 스트림, 로그 뷰어).
