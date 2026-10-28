# ADR 0014: 배포: 로컬 PC + Cloudflare Tunnel, pull 방식 배포

- 상태: 채택
- 날짜: 2026-10-28
- 대체: ADR 0010

## 맥락

ADR 0010은 Oracle Cloud Always Free ARM(2 OCPU / 12GB)을 골랐지만, Osaka 리전(가용성 도메인 1개)에서 인스턴스 생성이 며칠간 "Out of capacity"로 실패했다(shop-sim-infra#1). 종량제 전환은 용량을 보장하지 않고, CLI 재시도는 기약이 없다. 비용 0원 목표는 그대로다. 0010에 대비책으로 적어 둔 플랜 B를 본 구성으로 올린다.

## 결정

- **실행**: 로컬 PC(Apple Silicon, arm64)에서 docker compose로 staging·production을 돌린다. compose를 역할별 파일로 나누고 주소를 환경 변수로 받는 원칙(0010)은 그대로다.
- **외부 공개**: Cloudflare Tunnel(named tunnel). PC의 `cloudflared`가 Cloudflare로 나가는 연결을 맺어 공유기 포트를 열지 않는다. 고정 호스트명은 보유 도메인 `p-stack.dev`(이미 Cloudflare DNS 사용 중)의 하위 도메인을 쓴다.
- **배포**: CI(GitHub Actions)는 이미지를 빌드해 GHCR에 올리기까지만 한다. PC의 배포 에이전트가 새 이미지를 받아 교체한다(pull 방식). 에이전트 도구는 1단계에서 정한다.
- **이미지**: arm64로 빌드한다(PC와 같은 아키텍처).
- **부가 서비스**: 무료 구간(R2, GHCR, GitHub Actions, Sentry) 유지. 부하 생성은 같은 PC에서 돌리되, 측정 해석 시 자원 경합을 감안한다.

## 결과

- 추가 비용 없이(보유 도메인 사용) 메모리 예산(12GB 안팎)을 그대로 쓴다
- 용량 부족·무료 한도 변경·유휴 회수 같은 클라우드 무료 계정 위험이 사라진다
- 대가와 보완책
  - PC가 꺼지거나 잠들면 서비스와 시뮬레이터가 멈춘다 → 상시 가동 설정, 멈춤은 장애 실습 대상으로 포스트모템에 남긴다
  - 레포가 public이라 PC에 GitHub self-hosted runner를 두면 외부 PR이 PC에서 코드를 실행할 수 있다 → runner를 두지 않고 pull 방식으로 배포한다
  - 운영과 개발이 같은 기계라 자원을 나눠 쓴다 → 운영 compose에 메모리 상한을 걸고, 부하 테스트 결과는 같은 기계 조건으로 해석한다
  - 터널은 기본적으로 응답을 버퍼링한다 → SSE 응답은 `Content-Type: text/event-stream`을 보내야 스트리밍된다
  - 13~14단계 VM 분리·k3s는 로컬 k3d 또는 로컬 VM으로 실습한다
  - 모든 데이터는 재생성 가능하게(시드 + 시뮬레이터 + Postgres 백업) 유지한다

## 검토한 대안

- Oracle 종량제 전환: Always Free 한도 안이면 0원이지만 용량을 보장하지 않는다
- Oracle CLI 재시도 스크립트: 비용은 없지만 언제 생성될지 알 수 없어 1단계가 막힌다
- 유료 VM 1대(16GB): 상시 가동이 보장되지만 월 비용이 생긴다(0010 작성 시 Lightsail 서울 16GB 월 $84)
- Quick Tunnel(`trycloudflare.com`): 도메인이 필요 없지만 주소가 매번 바뀌고 SSE를 지원하지 않으며 테스트 용도다 → 웹훅·OAuth 콜백에 쓸 수 없다
- PC에 self-hosted runner로 push 배포: 설정이 단순하지만 public 레포에서 위험하다
