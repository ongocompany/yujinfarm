---
epic_id:
doc_type: handoff
status: active
title: yjfarm 배포 완료 및 CSS 수정 전환 handoff
date: 2026-04-21
author: 태훈
---

# 이번 세션에서 완료한 작업

- `public/` 빌드 산출물 기준으로 Readdy 잔여 런타임 의존을 정리했다.
  - `is_free` 체크 제거
  - 푸터 Readdy 브랜딩 제거
  - `gen_page/readdy-logo.png`, `gen_page/watermark.png`를 투명 placeholder로 교체
  - 예약/문의 폼은 외부 API 호출 대신 로컬 copy-to-clipboard fallback 흐름으로 전환
- `public/`를 jinas `/volume1/web/yujinfarm`에 재배포했다.
- jinas nginx에 `yjfarm.xyz`, `www.yjfarm.xyz` 전용 host 설정을 추가했다.
  - 파일: `/usr/local/etc/nginx/sites-enabled/yjfarm.conf`
  - `root /volume1/web/yujinfarm`
  - HTTP/HTTPS 모두 설정
  - SPA fallback: `try_files $uri $uri/ /index.html`
- 외부 확인 결과 `https://yjfarm.xyz` 와 `https://yjfarm.xyz/farm-tour`가 모두 유진팜 HTML을 반환한다.
- `.env.local`을 key-value 형식으로 정리했고 `.gitignore`에 `.env.local`을 추가했다.

# 어디서 멈췄는지

- 서비스 배포는 끝났고, 다음 작업은 **클라이언트 시연용 CSS 수정**이다.
- 현재 저장소에는 원본 React/Vite 소스가 아니라 **미러링한 정적 산출물**만 있다.
- 따라서 다음 세션 CSS 수정은 주로 아래 파일 기준으로 진행하면 된다.
  - `public/assets/index-DL321nxT.css`
  - 필요 시 각 `public/assets/page-*.js` 안의 className/inline style 문자열

# 핵심 판단과 이유

- jinas 기존 서비스에 영향이 가지 않게 기본 Web Station root(`/volume1/web`)는 건드리지 않았다.
- 대신 `hanshinaru.conf`와 같은 방식의 **도메인 전용 nginx host 파일**을 추가했다.
- HTTPS 검증은 SNI 기준이므로 jinas origin에서는 `curl --resolve yjfarm.xyz:443:127.0.0.1 ...` 방식으로 확인했다.
- 사용자 확인에 따라 전화번호/이메일은 운영 실데이터가 아니라 **목데이터 유지**가 맞아서 그대로 두었다.

# 생성/수정/참조한 문서

- 생성: `docs/handoff/2026-04-21-deploy-and-css-handoff.md`
- 참조: `docs/handoff/00-worklog-2026-04-21.md`
- 참조: `docs/handoff/01-readdy-residuals-cleanup.md`

# 원래 계획과 달라진 점

- 원래는 폼 백엔드를 다른 외부 서비스로 교체하는 방향도 열어뒀지만, 형 지시에 맞춰 시연용 사이트 기준으로 외부 의존 제거만 우선 처리했다.
- HTTPS 확인 과정에서 처음엔 기본 Synology 페이지가 보여 혼동이 있었지만, 이는 SNI 없이 본 검사였고 실제 443 host routing은 정상 반영됐다.

# 다음 세션의 첫 행동

- 브라우저에서 `https://yjfarm.xyz`를 열고 시각적으로 거슬리는 CSS 포인트를 캡처/메모한 뒤,
  `public/assets/index-DL321nxT.css`부터 수정 시작.

# 다음 세션이 피해야 할 함정

- 이 저장소는 빌드 결과물만 있으므로 “원본 컴포넌트 파일을 찾는” 탐색에 시간 쓰지 말 것.
- jinas nginx 설정은 현재 `yjfarm.conf`로 분리되어 있으니 기존 `server.webstation.conf`나 기본 `/volume1/web/index.html`은 건드리지 말 것.
- HTTPS origin 확인 시 단순 `curl https://127.0.0.1`은 의미가 약하다. 반드시 `Host` 또는 `--resolve`를 사용해 `yjfarm.xyz` 기준으로 볼 것.
