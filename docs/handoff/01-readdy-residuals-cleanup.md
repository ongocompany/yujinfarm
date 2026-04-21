# 업무지시서 01 — Readdy 잔여 의존 제거

- **작성일**: 2026-04-21
- **선행 작업**: 이미지 로컬화 완료 (`worklog-2026-04-21.md` 참조)
- **우선순위**: 높음 — Readdy 구독 해지 전 **반드시 완료**해야 할 항목
- **예상 소요**: 1~2 세션

---

## 1. 배경

Readdy.co 구독으로 생성된 SPA를 jinas에 셀프호스팅하는 작업 중, 정적 CDN 이미지
(`public.readdy.ai`, `static.readdy.ai`) 21개와 AI 생성 이미지
(`readdy.ai/api/search-image`) 87개는 로컬화 완료.

그러나 JS 번들에는 **런타임 기능**이 Readdy 서버에 의존하는 부분이 남아 있다.
구독을 해지하면 다음 기능이 깨진다:

| 종류 | 잔여 URL | 영향 |
|------|----------|------|
| 예약 폼 | `readdy.ai/api/form/d6200doomoleh5thvr10` | 예약 기능 오작동 |
| 문의 폼 | `readdy.ai/api/form/d61t8j3n6d2i4h4oae2g` | 문의 전송 실패 |
| 계정 체크 | `readdy.ai/api/public/user/is_free` | 불명 (클라이언트 분기 로직) |
| 로고·워터마크 | `readdy.ai/?ref=logo`, `readdy.ai?ref=b`, `/gen_page/readdy-logo.png`, `/gen_page/watermark.png` | 유료 해지 후에도 Readdy 브랜딩 노출 |

영향 파일: `public/assets/*.js` (minified). 원본 소스는 없고 wget 미러링 결과물만
보유 중이므로, 수정은 **번들 JS 안의 문자열·로직을 직접 치환**하는 방식으로 진행.

---

## 2. 작업 범위

### 2-1. 폼 백엔드 교체 (예약·문의 2건)

**현재 동작** — `assets/*.js` 안에 다음 패턴의 `fetch` 호출 2건 존재:

```js
await fetch("https://readdy.ai/api/form/<id>", { method: "POST", headers: ..., body: ... })
```

**교체 옵션 (택1, 장단점 비교)**

| 옵션 | 장점 | 단점 |
|------|------|------|
| **Formspree** (`formspree.io`) | 셋업 5분, 월 50건 무료, HTML form 호환 | 월 50건 초과 시 유료($10/월~) |
| **Google Forms** | 무료 무제한, 스프레드시트 자동 저장 | UI 이질감, POST payload 구조 맞추기 번거로움 |
| **자체 백엔드** (jinserver FastAPI / Synology PHP) | 완전한 통제, 기존 인프라 재활용 | 개발·운영 비용 |
| **Telegram Bot 알림** | 진우님께 바로 push 알림 가능, 무료 | 데이터 보관은 별도 필요 |

**권장**: **Formspree + Telegram 알림 이중화** — 데이터는 Formspree 대시보드에서
확인, 실시간 알림은 Telegram Bot webhook으로.

**체크리스트**
- [ ] 폼 백엔드 옵션 최종 결정 (사용자 확인)
- [ ] 폼 제출 시 전송되는 필드 스키마 파악 — `assets/*.js`에서 `append(` 호출 그레핑
- [ ] 신규 endpoint에 테스트 제출 → 정상 수신 확인
- [ ] JS 번들 내 두 URL을 `sed`로 신규 endpoint로 치환
- [ ] 로컬 preview에서 예약·문의 폼 실제 제출 테스트
- [ ] jinas 재배포 후 실서비스 제출 테스트
- [ ] 신규 endpoint의 CORS 설정 확인 (yujinfarm 도메인 허용)

---

### 2-2. `is_free` 체크 제거

**현재 동작** — JS 안에 다음 로직 존재:

```js
const s = "https://readdy.ai/api/public/user/is_free";
const i = "2680271b-d2fd-486a-8c...";  // projectId
// ... fetch 또는 hostname 체크
ey = () => {
  const s = window.location.hostname;
  return ["readdy.ai","dev.readdy.ai","localhost"].some(c => s === c || s.endsWith("."+c));
};
```

**영향**: `is_free` API는 구독 상태에 따라 무료 기능 제한을 강제하는 용도로 추정.
셀프호스팅 도메인에서 호출 시 동작 불명(에러 발생 또는 무료 모드 강제).

**권장**: 해당 `fetch` 호출과 `ey()` 관련 분기를 무력화 — `ey`를
`() => true`로, `ay`(호출 함수 추정)를 `() => {}` 또는 즉시 return으로 치환.

**체크리스트**
- [ ] `is_free` 호출 지점 정확한 파일·변수명 식별
- [ ] `fetch` 호출 제거 또는 `Promise.resolve()` 치환
- [ ] hostname 체크 배열에 셀프호스팅 도메인 추가 또는 함수 전체 무력화
- [ ] 브라우저 콘솔에 관련 오류 없는지 확인

---

### 2-3. Readdy 브랜딩 제거 (로고·워터마크·링크)

**현재 잔재**
- `public/gen_page/readdy-logo.png`, `public/gen_page/watermark.png` — 로컬화된 이미지 파일
- JS 안 `readdyLink: "https://readdy.ai?ref=b"`, `href: "https://readdy.ai/?ref=logo"` — 푸터 링크

**권장**
- 이미지 파일 2개 삭제 또는 투명 1×1 PNG로 교체
- JS 안 레퍼럴 링크를 `"#"` 또는 `"/"` 로 치환
- (선택) 사이트 푸터에 "유진팜앤스테이 © 2026" 등 자체 브랜딩 삽입 — 번들 JS 내부
  `<a>` 태그 문자열 치환

**체크리스트**
- [ ] JS 안 Readdy 링크 치환 (`sed`)
- [ ] 로고·워터마크 이미지 제거 여부 결정
- [ ] 푸터 렌더 확인 (Footer chunk: `assets/Footer-BviLzrls.js`)

---

## 3. 수용 기준 (Acceptance Criteria)

- [ ] `grep -r 'readdy\.ai' public/` 결과 **0건**
- [ ] 로컬 preview에서 예약·문의 폼 제출 → 신규 endpoint 수신 확인
- [ ] 브라우저 콘솔 에러·경고 0건 (폰트 404 제외)
- [ ] jinas 실서비스에서 폼 제출 종단간 동작 확인
- [ ] 푸터에 Readdy 브랜딩 노출 없음

---

## 4. 참고 자료

- JS 번들 분석 시 유용한 명령:
  ```bash
  cd public/assets
  grep -hoE 'fetch\("https?://[^"]+"' *.js | sort -u
  grep -hoE '"[a-zA-Z0-9_-]{15,}"' *.js | sort -u | head -30   # 프로젝트 ID 찾기
  ```
- 현재 JS 번들은 minified + Vite chunk hash — 번들 재생성 불가(원본 소스 없음),
  문자열 치환만 가능
- 치환 시 `sed -i ''` (macOS BSD) 사용, `.js` 확장자만 타겟팅
- 치환 후 반드시 `preview_start` + `window.location.reload()` + 네트워크 탭에서
  `readdy.ai` 요청 0건 확인
