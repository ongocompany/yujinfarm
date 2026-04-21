---
epic_id:
doc_type: handoff
status: active
title: CSS 디자인 시안 튜닝 (워커힐 톤 리디자인 draft)
date: 2026-04-21
author: 지훈
---

# 이번 세션에서 완료한 작업

1. 기존 사이트 CSS·JS 번들의 디자인 언어 스캔
   - 폰트 pairing: Noto Sans KR + Nanum Myeongjo가 JSX inline `fontFamily`로 8개 chunk에 박혀 있음 (entry·NotFound·notices·stay chunk는 inline 없음 → 시스템 폰트 fallback 위험)
   - 그린 11종 (#1B5E20 / #2E7D32 / #558B2F / #7CB342 / #A5D6A7 / #C5E1A5 / #C8E6C9 / #E8F5E9 / #F1F8E9 / #33691E / #689F38) + 주황·핑크·블루·인스타 3색 그라디언트까지 총 컬러 사용량 집계
   - rounded/shadow/font-size/gap/padding utility 사용 빈도 조사 (rounded 236, rounded-full 89, py-20 15, p-6 28, gap-8 19, mb-12 18 등 실데이터 기반)
   - **chunk 구조 재발견**: `Footer-BviLzrls.js`가 이름과 달리 Header + Footer 공통 Layout chunk. desktop nav도 여기. 다른 chunk에서 찾다 헤맸음 — 다음 세션에서 함정 주의.
   - inline JSX `style:{...}` 내 hex color 지정 **0건**, `borderRadius:"12px"` 1건만 — CSS override로 컬러/라운드 대부분 대응 가능 확인

2. `public/assets/index-DL321nxT.css` 끝에 4단계 append-only 튜닝 블록 추가 (원본 Tailwind utility는 한 글자도 수정 안 함 → revert 용이)

   - **(a) polish** — 한글 폰트 fallback 강화(Apple SD Gothic Neo / Malgun Gothic), `word-break: keep-all`, `:focus-visible` 접근성 복원(`.focus\:outline-none`을 `!important`로 override), `text-[9/10/11px]` → 11px 가독성 보정, `scroll-padding-top: 80px` + `prefers-reduced-motion` 존중, `::selection` 설정
   - **(b) walker-hill tone 리디자인** — 컬러 greyscale + single accent `#3F4E3C` (moss/deep olive). 그린 11종과 주황·핑크·블루·보라 계열을 모두 accent 또는 grey(`#1A`, `#2A`, `#5A`, `#6B`, `#D9`, `#E5`, `#EE`, `#F5`, `#FAFAFA`)로 재매핑. `from-*`/`via-*`/`to-*` gradient stop도 동일 체계로. 모든 border-radius 0 (rounded-full 89회는 유지). shadow 한 단계씩 완화 + 차콜 rgb. 폰트 위계 4단으로 압축(Display 4rem / Title 2rem / Body 0.94rem / Caption 0.75rem). body 색 `#1A1A1A`. `::selection` accent로 재정의. 일반 링크 hover만 accent, 기본은 inherit.
   - **(c) density pass** — 섹션 vertical (py-24→3.5rem, py-20→3rem, py-12→2rem 등), 박스 내부 (p-12→2rem, p-8→1.5rem, p-6→1.25rem 등), gap-12/8/6 축소, mb-16/12 축소, space-y-8/6 축소. 버튼 내부 `py-3` 이하·`gap-4` 이하는 답답함 방지로 보존.
   - **(d) nav pass** — desktop `lg:flex` 상단 nav(Footer chunk 내): 메뉴 버튼 `text-sm`(density pass 후 13px) → **15px**, hover underline bar **5px → 2px**, 메뉴 텍스트와 바 사이 **padding-bottom 8px** gap. Selector는 `.hidden.lg\:flex.items-center.justify-between` 전체 class 조합으로 고유성 확보.

3. 반응형 커버리지 검증
   - polish / 컬러 / 라운드 / 그림자 / body color / selection / 링크 hover = **전 뷰포트 공통**
   - 폰트 위계 = `base` + `@media(min-width:768px){.md\:text-*}` + `@media(min-width:1024px){.lg\:text-6xl/7xl}` 각자의 scope 안에서 override
   - density pass = `base` + `@media(min-width:768px){.md\:py-*, .md\:p-*, .md\:gap-*, .md\:mb-*, .md\:mt-*, .md\:space-y-*}`
   - nav pass = **desktop(`lg:flex`) 전용** (모바일은 햄버거 드로어 구조라 별개)

4. advisor 검토 2회 반영
   - 1회: `md:`/`lg:` variant override를 원본 `@media` scope 안으로 이동하지 않으면 모바일 레이아웃 깨짐 — 블로커 해결
   - 2회: 헤딩 serif(Nanum Myeongjo) 강제는 JSX inline `'Noto Sans KR'`가 헤딩 대부분에 박혀 있어 거의 효과 없음 — Noto Sans로 통일하고 자간·스케일로만 세련미 확보하는 방향으로 전환

5. 로컬 `http://localhost:5174/` Python http.server 기반 프리뷰로 CSS 응답 확인
   - 최종 CSS 크기 39,354 → **54,326 chars** (4개 블록 총 ~15KB append)
   - 주석 제거 후 `{}` 774=774, `()` 769=769 — 구문 균형
   - `readdy.ai` 참조 regression 0건 유지

# 어디서 멈췄는지

- **시안 단계**. 본안 결정 전까지 `/codex review`는 보류 (CLAUDE.md의 100줄+ 룰에도 불구하고 진우형 지시).
- 로컬 검증 끝. jinas 수동 배포 실행 직전.
- 모바일 nav 현 상태 유지 (hover 개념 없음, drawer 메뉴 글자 15px, 상단 로고 16px로 적절).

# 핵심 판단과 이유

- **append-only 원칙** — 시안이라 원본 Tailwind utility 미수정. git revert 시 CSS 한 파일만 되돌리면 즉시 롤백 가능.
- **accent `#3F4E3C` (moss/deep olive)** — 진우형이 제시된 3안 중 선택. 자연 친화 정체성 + 워커힐 톤 중간 지점.
- **border-radius 완전 0 (rounded-full 제외)** — 진우형의 "AI가 만든 디자인 티" 지적 직결. 워커힐 벤치마크도 sharp 90° dominant.
- **폰트 위계 4단 압축** — "배리에이션 너무 많아 세련 부족" 대응. Display / Title / Body / Caption.
- **md:/lg: variant override를 @media scope 안에 배치** — 바깥에서 덮으면 모바일에 desktop 값이 강제 적용돼 레이아웃 깨짐. advisor 블로커 지적이 결정적.
- **nav underline gap은 padding-bottom + height 축소** 로 해결. Bar position(`absolute bottom-0`)을 건드리지 않고 container 높이를 키워 bar가 자연스럽게 아래로 밀려남.
- **이미지 위 black 오버레이 그라디언트는 미건드림** — 기능적(텍스트 가독성)이라 컬러 재매핑에서 의도적 제외.

# 생성 / 수정 / 참조한 문서

- 생성: `docs/handoff/2026-04-21-css-redesign-draft.md` (본 문서)
- 수정: `public/assets/index-DL321nxT.css` (39,354 → 54,326 chars, 4개 append-only 블록 추가)
- 참조: `docs/handoff/00-worklog-2026-04-21.md`, `docs/handoff/01-readdy-residuals-cleanup.md`, `docs/handoff/2026-04-21-deploy-and-css-handoff.md`

# 원래 계획과 달라진 점

- 초기는 "Option B (Conservative 톤업 + 헤더 폴리쉬)"로 가려 했으나 진우형 피드백으로 리디자인 스코프가 **워커힐 톤 대폭 그레이스케일 + 밀도 UP + nav 손봄**으로 확장.
- 초반 `h1/h2` serif(Nanum Myeongjo) 강제 계획은 advisor 검토로 기각 → sans-serif 통일로 전환.
- `backdrop-filter` 헤더 glass는 배경 불투명이면 no-op이라 riskless로 유지.
- `/codex review`는 본안 확정 후로 이연.

# 다음 세션의 첫 행동

1. 로컬 `http://localhost:5174/` + 실서비스 `https://yjfarm.xyz` 양쪽에서 5 라우트 (`/`, `/farm-tour`, `/stay`, `/programs`, `/news`) × 3 뷰포트 (375 / 768 / 1280) 최종 시각 검증
2. 본안 결정 후 `/codex review` 실행 → 치명 이슈만 반영
3. (필요 시) 모바일 nav 추가 튜닝 pass
4. 본안 확정 후 append-only 4블록을 단일 "brand redesign" 블록으로 통합·정리 (CSS 위생 작업)
5. **Cloudflare 캐시 정책 정식 정리** — 현재 `/assets/*`에 `Cache-Control: public, max-age=31536000, immutable` 헤더가 붙어 있어, CSS 파일 내용만 바꾸는 iteration 방식에서 edge/브라우저가 revalidate하지 않음. 시안 단계에선 `index.html`의 CSS 링크에 `?v=20260421c` 식 쿼리 버전을 증가시키는 방식으로 우회 중. 본안 단계에서 Cloudflare Page Rule로 `immutable` 제거 또는 TTL 단축, 혹은 Vite 재빌드 기반 파일명 해시 convention 복원 중 택일.

# 다음 세션이 피해야 할 함정

- **Header chunk는 `Footer-BviLzrls.js`** — 이름 혼동 주의. desktop nav 문자열 전부 이 파일에.
- `md:`/`lg:` variant override는 반드시 원본 `@media` scope 내부에 배치. 바깥에서 덮으면 모바일 `grid-cols-*`/`text-*`/`py-*`가 모바일 뷰포트에 강제 적용돼 레이아웃 깨짐.
- append-only 원칙 — 원본 Tailwind utility를 직접 수정하지 말 것. cascade 순서·specificity 꼬임 방지.
- inline JSX `style={{...}}`로 박힌 속성은 CSS로 못 덮음 — 다행히 현 번들 `style:{...}` 안에 hex color 지정 0건, `borderRadius:"12px"` 1곳만 남음. 향후 수정 시 재확인.
- `rounded-full`(89회)은 반드시 유지 — 아바타·뱃지·아이콘 원형성 필수.
- `/news#notices-section` 같은 in-page anchor 동작은 `scroll-padding-top: 80px`에 의존. 헤더 실제 높이 변화 시 값 조정 필요.
- `.hidden.lg\:flex.items-center.justify-between` 이 유일한 desktop nav selector임을 기억. 다른 chunk에 같은 조합 없음 확인했으나, 차후 추가되면 의도치 않은 override 가능.
