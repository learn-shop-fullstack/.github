# ADR 0019: SPA 빌드 도구: Vite

- 상태: 채택
- 날짜: 2026-10-30

## 맥락

셀러센터·어드민은 같은 디자인 시스템·그리드 패키지를 공유하는 React SPA로 정해져 있다(shop-sim-web 결정사항). 빌드 도구는 첫 SPA 항목(1단계 셀러센터·어드민 빈 레이아웃)으로 미뤄 두었다.

- web 레포는 pnpm + Turborepo 모노레포이고, 내부 패키지는 빌드 없이 TS 소스를 내보내 앱의 번들러가 변환한다.
- 테스트는 Vitest 5다. Vitest는 Vite 위에서 돌고, 레포에는 이미 Vite 8.3.5가 Vitest를 통해 설치돼 있다.
- API 주소는 빌드할 때 프레임워크 관례대로 주입하고 환경마다 이미지를 따로 빌드한다(ADR 0020).

## 결정

- **도구**: Vite 8(정확한 버전 고정)과 `@vitejs/plugin-react`. 셀러센터(`apps/seller`)와 어드민(`apps/admin`)이 같은 구성을 쓴다.
- **API 주소**: Vite 관례대로 `VITE_API_BASE_URL`을 빌드할 때 `import.meta.env`로 넣는다(ADR 0020). `packages/api-client`가 이 이름을 읽는다.

## 결과

- 테스트(Vitest)와 빌드가 같은 Vite 설정·플러그인·변환을 쓴다. Vite 버전은 Vitest가 요구하는 범위와 함께 올린다
- React SPA의 사실상 표준이라 AG Grid·Tiptap·차트 라이브러리 문서와 예제를 그대로 쓸 수 있다
- 대가와 보완책
  - 메이저 버전마다 번들러 내부가 바뀐다(8에서 Rolldown으로 교체). 0.x가 아니어도 업그레이드는 별도 PR로 빌드 결과물 크기와 dev 서버 동작을 확인한다
  - `VITE_` 접두어가 붙은 변수는 모두 번들에 들어간다 → 공개해도 되는 값만 이 접두어로 둔다

## 검토한 대안

- **Rsbuild 2.3.0(Rspack)**: webpack 호환 API에 빌드가 빠르다. 그러나 생태계가 작고, 테스트 도구(Vitest)와 계열이 달라 설정과 변환을 두 벌 관리한다
- **Next.js static export**: 스토어와 도구가 하나가 된다. 그러나 SPA로 정한 성격과 섞이고(서버 기능을 쓸 수 없는 Next), 동적 라우트를 정적으로 내보내는 데 제약이 있어 관리 화면의 상세 경로마다 우회가 필요하다
