# ADR 0021: SPA 라우터: TanStack Router

- 상태: 채택
- 날짜: 2026-10-31

## 맥락

셀러센터·어드민(React SPA, Vite, ADR 0019)의 라우터는 첫 SPA 항목(1단계 빈 레이아웃)으로 미뤄 두었다.

- 관리 화면은 그리드의 검색어·필터·정렬·페이지를 URL 쿼리에 둔다. 새로 고침·공유·뒤로 가기에 상태가 남아야 하고, 잘못된 쿼리(손으로 고친 URL)를 받아도 화면이 깨지면 안 된다.
- 서버 상태는 TanStack Query로 다루고 쿼리 옵션은 `packages/api-client`가 생성한다(ADR 0017). 화면 진입 전에 데이터를 미리 받는 지점이 라우터에 필요하다.
- SPA라 서버 렌더링과 서버 데이터 함수는 쓰지 않는다.

## 결정

- **라우터**: TanStack Router(정확한 버전 고정). 셀러센터(`apps/seller`)와 어드민(`apps/admin`)이 같은 구성을 쓴다.
- **라우트 정의**: 파일 기반. Vite 플러그인(`@tanstack/router-plugin`)이 라우트 파일에서 라우트 트리를 생성하고, 라우트마다 코드를 나눈다(`autoCodeSplitting`).
- **URL 쿼리**: 라우트의 `validateSearch`로 검증하고 기본값을 채운다. 화면 코드는 검증된 타입으로만 읽는다. 검증 라이브러리는 첫 사용처에서 정한다.
- **데이터**: 라우터 context로 `QueryClient`를 넘기고, 라우트 `loader`에서 api-client 쿼리 옵션으로 `ensureQueryData`·`prefetchQuery`를 부른다. 캐시는 TanStack Query가 맡는다(라우터 자체 캐시에 기대지 않는다).

## 결과

- 경로 파라미터, URL 쿼리, 링크 대상이 타입으로 검사된다. 없는 경로나 빠진 파라미터는 typecheck에서 걸린다
- 스토어(서버 컴포넌트 prefetch)와 SPA(loader prefetch)가 같은 api-client 쿼리 옵션을 쓴다
- 대가와 보완책
  - 라우트 트리 생성 파일이 생긴다. 생성 전에는 typecheck가 실패한다 → api-client 생성물처럼 다루고, typecheck·build 전에 생성되게 turbo 태스크를 맞춘다(구현 이슈에서 정한다)
  - React Router보다 생태계와 자료가 작다 → 공식 문서와 예제 범위 안에서 쓰고, 막히면 이 ADR을 다시 본다
  - 1.x에서도 마이너 버전이 자주 나온다 → 정확한 버전으로 고정하고 업그레이드는 별도 PR로 한다

## 검토한 대안

- **React Router 8 data mode(`createBrowserRouter` + `loader`)**: 사용자가 가장 많고 자료가 풍부하다. 그러나 경로 타입 생성은 framework mode용이라 이 모드에서는 params·URL 쿼리 타입과 검증을 직접 만들어야 한다
- **React Router 8 framework mode(SPA, `ssr: false`)**: 경로 타입을 생성해 준다. 그러나 React Router 전용 Vite 플러그인과 앱 구조가 빌드를 맡아 ADR 0019의 기본 Vite 구성에서 벗어나고, URL 쿼리 검증은 여전히 없다
- **TanStack Router 코드 기반 라우트**: 생성 파일이 없다. 그러나 라우트 트리를 손으로 이어야 하고 라우트별 코드 분할도 직접 해야 한다
