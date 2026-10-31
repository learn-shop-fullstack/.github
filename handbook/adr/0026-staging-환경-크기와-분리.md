# ADR 0026: staging 환경: 상한을 줄여 상시 운영하고 기능 확인에만 쓴다

- 상태: 채택
- 날짜: 2026-10-31

## 맥락

staging·production은 같은 PC의 별도 compose 프로젝트다(ADR 0014). 프로젝트 이름은 `shop-sim-<환경>`, 비밀 env·배포 상태 파일·터널이 환경마다 따로 있다(ADR 0022, 0023, 0025).

- PC는 32GB이고 컨테이너는 OrbStack VM(15.7GiB) 안에서 돈다
- 운영 스택(data + app + cloudflared)의 메모리 상한 합계가 5.95GiB다(shop-sim-infra#8). staging을 같은 구성으로 띄우면 약 11.9GiB다
- 12GB 메모리 예산(architecture.md)은 VM 한 대에 한 환경을 올리던 ADR 0010 시절의 기준이라 staging이 들어 있지 않다. 관측성(1.5GB, 13단계), 개발 스택, 부하 생성도 같은 VM을 쓴다

## 결정

- **용도**: staging은 배포가 되는지와 기능이 맞는지를 확인하는 곳이다. 성능·부하 측정은 production에서 한다.
- **크기**: staging은 상한을 줄인 구성으로 상시 띄운다. 목표 합계는 약 3.3GiB다.

  | 서비스 | production | staging |
  |---|---|---|
  | Postgres | 1GiB | 384MiB |
  | Redis | 200MiB (maxmemory 150mb) | 64MiB (48mb) |
  | Kafka | 1GiB (힙 512m) | 640MiB (힙 256m) |
  | OpenSearch | 2GiB (힙 768m) | 1.25GiB (힙 384m) |
  | api·worker·store | 각 512MiB | 각 256MiB |
  | seller·admin·cloudflared | 64·64·128MiB | 같음 |

- **방식**: 공용 compose 파일은 production 값을 갖고, staging은 환경별 덮어쓰기 파일(`compose.staging.yml`)로 상한과 힙만 바꾼다. 서비스 구성·이미지·설정은 같다. 배포 에이전트가 환경별 파일이 있으면 함께 넘긴다.
- **분리**: 볼륨·네트워크·컨테이너는 프로젝트 이름으로 나뉘고 서로 공유하지 않는다. 데이터 서비스도 환경마다 따로 띄운다.
- **예산**: VM 안의 상한 합계를 production 12GB 예산 + staging 약 3.3GiB로 본다. 개발 스택은 개발할 때만 띄우고 이 합계에 넣지 않는다.

## 결과

- 두 환경이 함께 상시로 돌면서 VM 한도 안에 들어온다
- staging에서 배포 절차(이미지, 마이그레이션, 터널)를 production과 같은 경로로 먼저 확인한다
- 대가와 보완책
  - staging의 힙과 상한이 production과 달라 메모리 관련 문제(OOM, GC)는 staging에서 재현되지 않거나 staging에서만 난다 → staging 결과로 성능을 판단하지 않는다. 메모리 문제는 production 수치로 본다
  - 줄인 값이 실제로 버티는지는 아직 모른다(특히 OpenSearch 384m 힙) → 구현에서 staging 데이터 스택을 띄워 대기 메모리를 재고, 모자라면 이 표를 고친다(1단계 메모리 실측 항목)
  - 덮어쓰기 파일이 공용 파일과 어긋날 수 있다(서비스 이름 변경 등) → 덮어쓰기에는 상한·힙만 두고, `config`로 두 환경을 함께 검사한다
  - 상한 합계는 사용량이 아니라 최댓값이다. 모두 동시에 상한까지 쓰는 일은 드물지만 VM 한도를 넘으면 OOM이 난다 → 개발 스택을 띄울 때 staging을 잠시 내리는 것을 허용한다

## 검토한 대안

- **staging을 production과 같은 구성으로**: 차이가 없다. 그러나 두 환경만으로 VM의 4분의 3을 쓰고 관측성·개발 스택이 들어갈 자리가 없다
- **staging은 필요할 때만 띄운다**: 구성 차이가 없다. 그러나 에이전트가 staging을 자동 적용하지 못해 GitOps 흐름이 staging에서 끊기고, 켜고 끄는 수작업이 생긴다
- **VM 메모리를 늘린다(20GiB 이상)**: 두 환경을 같은 구성으로 둘 수 있다. 그러나 맥 본체 메모리가 줄고 "예산 안에서 설계한다"는 원칙이 느슨해진다
- **staging이 production 데이터 서비스를 공유(DB·토픽·인덱스 이름만 분리)**: 메모리가 가장 적다. 그러나 staging의 실수(마이그레이션, 대량 이벤트)가 production 데이터와 성능에 닿는다
