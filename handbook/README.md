# shop-sim 엔지니어링 핸드북

멀티 판매자 마켓플레이스 **shop-sim**을 회사처럼 개발·운영하는 학습 프로젝트의 공통 문서다.

| 문서 | 내용 |
|---|---|
| [working-agreement.md](working-agreement.md) | Claude와의 협업 방식, 이슈·브랜치·커밋·PR·ADR·배포·운영 절차 |
| [architecture.md](architecture.md) | 시스템 구성, 레포 구성, 계약, 배포 환경, 도메인 모델, 결제 구성, 주문 상태 |
| [ROADMAP.md](ROADMAP.md) | 0~14단계 작업 목록 (레포 태그별) |
| [adr/](adr/) | 아키텍처 결정 기록 |

## 레포

| 레포 | 성격 |
|---|---|
| `.github` | 이 핸드북, 조직 공통 PR·이슈 템플릿 |
| `shop-sim-api` | 백엔드 (NestJS) |
| `shop-sim-web` | 프론트엔드 (스토어, 셀러센터, 어드민) |
| `shop-sim-infra` | 인프라·배포·관측성 |
| `shop-sim-vendor` | 외부 회사 흉내 (모의 PG, 모의 택배사) |
| `shop-sim-loadtest` | 트래픽 시뮬레이터, 부하 테스트 |

## 로컬 작업 공간

모든 레포를 한 폴더 아래에 클론한다. 각 레포의 CLAUDE.md가 `../.github/handbook/`의 문서를 불러오므로 이 배치를 지켜야 한다.

```
learn-fullstack/        (git 레포 아님, 작업 공간)
  .github/
  shop-sim-api/
  shop-sim-web/
  shop-sim-infra/
  shop-sim-vendor/
  shop-sim-loadtest/
```

한 레포 안의 작업은 그 레포에서, 여러 레포에 걸친 작업(스펙 변경 + 구현 + 소비자 반영)은 `learn-fullstack/`에서 Claude Code를 연다.
