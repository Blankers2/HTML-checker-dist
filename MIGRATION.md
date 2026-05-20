# HTML Checker — 마이그레이션 & 개발자 가이드

> 이 문서는 HTML Checker를 **사내 환경에 도입**하거나 **고객사 워크플로우에 통합**하려는 개발자·운영자·기술 PM을 위한 완전 가이드입니다.
>
> v1.0 (2026-05) 기준 · 약 2,478 라인 단일 파일 · 의존성 0개

---

## 목차

0. [한눈에 보기 (TL;DR)](#0-한눈에-보기-tldr)
1. [요구사항 (Requirements)](#1-요구사항-requirements)
2. [도입 방식 — 3가지 옵션](#2-도입-방식--3가지-옵션)
3. [아키텍처 개요](#3-아키텍처-개요)
4. [데이터 흐름 — postMessage Bridge 프로토콜](#4-데이터-흐름--postmessage-bridge-프로토콜)
5. [핵심 모듈 매핑](#5-핵심-모듈-매핑)
6. [커스터마이징 포인트](#6-커스터마이징-포인트)
7. [파일명 규약 (Naming Convention)](#7-파일명-규약-naming-convention)
8. [배포 가이드](#8-배포-가이드)
9. [브라우저 호환성 & 보안](#9-브라우저-호환성--보안)
10. [퍼포먼스 & 한계](#10-퍼포먼스--한계)
11. [확장 시나리오](#11-확장-시나리오)
12. [트러블슈팅](#12-트러블슈팅)
13. [용어집](#13-용어집)

---

## 0. 한눈에 보기 (TL;DR)

| 항목 | 내용 |
|---|---|
| **무엇** | 마케팅 HTML 번역(EN/JA) 파일을 한국어 원본과 함께 비교·검수하는 브라우저 기반 QA 도구 |
| **누가** | HTML 콘텐츠 검수자, 번역 QA 인력, 콘텐츠 운영팀 |
| **언제** | 다국어 HTML이 외주/번역 업체로부터 납품되어 검수가 필요할 때 |
| **어디** | 정적 호스팅이 가능한 어떤 환경이든 (GitHub Pages, S3, CDN, Nginx) |
| **어떻게** | `index.html` 하나만 배포하면 끝. 별도 설치·계정·백엔드 불필요 |
| **왜** | 수작업 2시간 → 15분 (체감 기준), 휴먼 에러 ↓, 검수 일관성 ↑ |

### 가장 빠른 도입 (60초)
```bash
# 1. 단일 파일 다운로드
curl -O https://raw.githubusercontent.com/Blankers2/HTML-checker-dist/main/index.html

# 2. 사내 서버 어디든 업로드 (예: Nginx html 디렉토리)
scp index.html user@internal.server:/var/www/qa-tools/

# 3. 브라우저로 접속
# https://internal.server/qa-tools/index.html
```

→ 끝. 사용자는 EN/JA/KR HTML 파일을 드래그앤드롭만 하면 됨.

---

## 1. 요구사항 (Requirements)

### 1.1 클라이언트 요구사항
| 항목 | 최소 | 권장 |
|---|---|---|
| 브라우저 | Chrome 90+ / Edge 90+ | 최신 Chromium 계열 |
| 메모리 | 2 GB RAM | 4 GB+ |
| 해상도 | 1280×720 | 1920×1080+ (2분할 비교 최적) |
| 키보드 | 일반 키보드 | 백틱(`)·F8·Esc 사용 가능한 키보드 |

> **Safari/Firefox 동작**: 대부분 작동하지만, iframe 렌더링이 Chrome과 다를 수 있어 검수 결과 신뢰성에 영향. **운영 환경은 Chromium 권장**.

### 1.2 서버 요구사항
- **정적 파일 호스팅** 1대로 충분 (백엔드 없음)
- HTTPS 권장 (Chrome의 일부 보안 정책상 file:// 프로토콜은 제약 있음)
- 트래픽 비용 거의 무시 가능 (`index.html` ≈ 130KB)

### 1.3 사용자 요구사항
- HTML 검수 업무 경험자 (마크업 구조에 대한 기본 이해)
- 브라우저 사용 능력 (드래그앤드롭, 키보드 단축키)
- 한국어 + 영어 + 일본어 기본 식별 가능 (검수 대상 언어이므로)

### 1.4 인풋 파일 요구사항
- HTML 파일 (UTF-8 인코딩 권장)
- `.text-info` 클래스를 가진 텍스트 블록 단위 (Flitto HTML 템플릿 기준)
- 파일명에 언어 식별 가능한 패턴 포함 (`_en_`, `_ja_`, `_origin_`)

---

## 2. 도입 방식 — 3가지 옵션

### 옵션 A. 사내 정적 서버 호스팅 (권장)
**대상**: 내부 사용만, 보안·접근 통제 필요한 경우
```nginx
# Nginx 예시
server {
  listen 443 ssl;
  server_name qa.internal.flitto.com;

  location /html-checker/ {
    alias /var/www/html-checker/;
    autoindex off;
    auth_basic "QA Tool";              # 필요시 베이직 인증
    auth_basic_user_file /etc/.htpasswd;
  }
}
```
- 장점: 사내망 보안, 접근 로그 관리 가능
- 단점: 운영 부담 (서버 1대 관리)

### 옵션 B. GitHub Pages / Object Storage (가장 간단)
**대상**: 빠른 배포, 외부 협업사도 접근 필요
- GitHub Pages: 무료, 자동 HTTPS, push만 하면 1~2분 내 반영
- AWS S3 + CloudFront: 비용 거의 0원, 사내망 우회 필요시 적합
- 사내 인증이 필요하면 S3 + Cognito 인증 게이트웨이

### 옵션 C. Electron / WebView 임베드
**대상**: 데스크톱 앱처럼 배포하고 싶을 때
```bash
# Electron 래퍼 예시 (5분이면 가능)
npm i electron
# main.js에서 BrowserWindow.loadFile('index.html')
electron-builder --win nsis --x64
```
- 장점: 인터넷 없이도 사용 가능, 시작프로그램 등록 가능
- 단점: 빌드/배포 오버헤드, 업데이트 채널 별도 필요

---

## 3. 아키텍처 개요

### 3.1 단일 파일 구조
```
index.html (≈ 2,478 lines)
├── <head>
│   ├── <meta> / <title>
│   └── <style> ─ 모든 CSS (≈ 1,000 lines)
│       ├── reset / variables / theme
│       ├── layout (header / panels / sidebar)
│       ├── minimap markers (search 10색 + korean)
│       └── animations (pulse / drop / fade)
│
├── <body>
│   ├── <header> ─ 좌측 파일 라벨 / 중앙 동기화·검색 / 우측 추출·테마
│   ├── <aside class="sidebar"> ─ TM 다중검색 전용
│   ├── <main class="split-view"> ─ EN / JA / KR 패널
│   └── <div id="toast">
│
└── <script> ─ 모든 JS (≈ 1,400 lines)
    ├── DOM 유틸 ($, sendMsg)
    ├── BRIDGE_SCRIPT (iframe 주입용 — toString으로 직렬화)
    ├── 파일 로드 / Blob URL / drag&drop
    ├── 스크롤 동기화 (mirror sync / one-shot / KR)
    ├── 미니맵 마커 (updateMinimapPM / clearMinimap / unmatched)
    ├── 한국어 자동 검출 (detectKorean)
    ├── TM 다중검색 (doSearch / pill 필터링 / 히스토리)
    ├── KR 원본 비교 (백틱 토글 / 위치 계산)
    └── 헤더 드롭다운 (자동 노출 / 고정 모드)
```

### 3.2 두 개의 실행 컨텍스트
이 앱의 모든 비밀은 "**parent와 iframe이 별도 컨텍스트**"라는 점에 있습니다.

```
┌─────────────────────────────────────────────────────────────┐
│  PARENT (index.html)                                        │
│  ┌────────┐  ┌────────┐  ┌────────┐                         │
│  │ Header │  │ Sidebar│  │ Toast  │  ← UI                   │
│  └────────┘  └────────┘  └────────┘                         │
│  ┌────────────────┐  ┌────────────────┐                     │
│  │ EN iframe      │  │ JA iframe      │  ← Blob URL 렌더    │
│  │ ┌────────────┐ │  │ ┌────────────┐ │                     │
│  │ │BRIDGE_SCRIPT│ │  │ │BRIDGE_SCRIPT│ │  ← 검색/스크롤    │
│  │ └────────────┘ │  │ └────────────┘ │     /검출 로직      │
│  └────────────────┘  └────────────────┘                     │
│                                                              │
│    ↕ postMessage ('search' | 'korean' | 'setScroll' | ...)  │
└─────────────────────────────────────────────────────────────┘
```

- iframe 내부: 검색/검출 등 **무거운 DOM 작업**을 격리
- parent: UI 상태와 **미니맵 마커**만 관리
- 통신: 모든 명령/응답은 `postMessage`로

이 구조 덕분에 EN iframe과 JA iframe이 동시에 검색을 실행해도 서로 간섭이 없고, 한쪽이 느려도 다른 쪽 UI는 멈추지 않습니다.

---

## 4. 데이터 흐름 — postMessage Bridge 프로토콜

### 4.1 메시지 타입 일람

#### Parent → iframe (명령)
| `m.t` | 페이로드 | 동작 |
|---|---|---|
| `search` | `{ queries: [{q, cls, idx}], opts }` | 다중 검색어 하이라이트 |
| `korean` | `{ opts? }` | 한국어 텍스트 자동 검출 |
| `extract` | `{}` | `.text-info` 블록 텍스트 추출 |
| `setScroll` | `{ top: number }` | 절대 픽셀 위치로 스크롤 |
| `scrollBy` | `{ dy: number }` | 상대 픽셀 스크롤 |
| `goBlock` | `{ i: number }` | `.text-info` 인덱스 i로 점프 |
| `clear` | `{}` | 검색 마크 모두 제거 |
| `getH` | `{}` | 현재 스크롤 높이 응답 요청 |

#### iframe → Parent (응답/이벤트)
| `m.t` | 페이로드 | 의미 |
|---|---|---|
| `searchR` | `{ id, r: [{ i, sn, count, top, qIdx }] }` | 검색 결과 |
| `koreanR` | `{ id, r: [{ i, top, text }] }` | 한국어 발견 위치 |
| `extractR` | `{ id, r: [string] }` | 추출된 텍스트 배열 |
| `scroll` | `{ top, h, c }` | 스크롤 변화 알림 (상시) |

### 4.2 ID 기반 요청-응답 매칭
```javascript
// parent 측
let _msgId = 0; const pending = new Map();
async function sendMsg(lang, payload) {
  const id = ++_msgId;
  return new Promise(resolve => {
    pending.set(id, resolve);
    $(lang + 'Frame').contentWindow.postMessage({ ...payload, id }, '*');
  });
}
// iframe 응답 시 pending.get(id) 호출
```

이 패턴 덕분에 한 iframe에 여러 명령을 동시 발사해도 응답이 섞이지 않습니다.

### 4.3 보안 모델
- iframe은 **`Blob URL`로 로드** → cross-origin은 아니지만 별도 origin 취급
- `postMessage`의 `targetOrigin`은 `'*'` 사용 (사내 도구 가정)
- **외부 배포 시 주의**: 사용자가 로드한 HTML에 악성 스크립트가 있어도 iframe sandbox로 격리되지만, 완벽한 안전을 위해서는 `sandbox="allow-scripts"` 속성을 iframe에 명시할 것

---

## 5. 핵심 모듈 매핑

> 함수 위치(라인 번호)는 `index.html` v1.0 기준 ± 50줄 오차 가능.

### 5.1 파일 로드 & 분류
| 기능 | 함수 | 라인 |
|---|---|---|
| 드래그앤드롭 분류 | `setupDrop()` + `detectLang()` | ~1720 |
| Blob URL 생성 + iframe 주입 | `loadHTML(html, lang)` | ~1620 |
| KR 이미지 CSS 보정 | inline `<style>` 주입 | ~1655 |

### 5.2 스크롤 동기화
| 기능 | 함수 | 라인 |
|---|---|---|
| 상시 동기화 (EN ↔ JA) | `scroll` 메시지 핸들러 + `setScroll` 전파 | ~1860 |
| 한 번 위치 동기화 | `syncNowBtn.onclick` | ~2100 |
| KR 단방향 동기 | `repositionKr()` + 활성 측 scroll 리스너 | ~2580 |

### 5.3 검색 & 미니맵
| 기능 | 함수 | 라인 |
|---|---|---|
| 다중 검색 실행 | `doSearch()` | ~2400 |
| 미니맵 마커 생성 | `updateMinimapPM(lang, results, type)` | ~2127 |
| 미니맵 클리어 (타입별) | `clearMinimap(type)` | ~2144 |
| Pill 필터링 + dim | `renderSearchPills()` + CSS `data-search-filter` | ~2460 |
| Unmatched 표시 (▶) | `applyUnmatchedMarkers(qIdx)` | ~2160 |

### 5.4 한국어 검출
| 기능 | 함수 | 라인 |
|---|---|---|
| iframe 측 한글 정규식 | `KO = /[가-힣]+/g` (bridge 내부) | bridge ~1390 |
| parent 측 결과 처리 | `detectKorean(opts)` | ~2170 |
| 드롭다운 자동 노출 | `detectKorean()` 끝 부분 분기 | ~2235 |
| 위치확인 버튼 | `koreanRescanBtn.onclick` | ~2270 |

### 5.5 KR 원본 비교
| 기능 | 함수 | 라인 |
|---|---|---|
| 백틱 토글 | `document.addEventListener('keydown')` | ~2650 |
| KR overlay show/hide | `showKrOverlay()` / `hideKrOverlay()` | ~2600 |
| 위치 재계산 | `repositionKr()` | ~2580 |

### 5.6 검색 히스토리
| 기능 | 함수 | 라인 |
|---|---|---|
| 저장 | `pushHistory(rows)` | ~2310 |
| 로드 | `loadHistory()` | ~2305 |
| 적용 | `applyHistory(snap)` | ~2335 |
| localStorage key | `SEARCH_HISTORY_KEY = 'oy-search-history'` | ~2295 |

---

## 6. 커스터마이징 포인트

### 6.1 색상 팔레트 변경 (다중 검색)
```javascript
// index.html 라인 ~2280
const SEARCH_COLORS = [
  { name: '노랑', hex: '#ffaa00' },
  // ... 10개
  { name: '검정', hex: '#1f2937', dark: true },  // dark 플래그로 대비 처리
];
```
변경 시 동시에:
- CSS의 `.mark-search-N { background: ... }` (라인 ~480)
- 브리지 CSS의 `mark.oy-search-hl-N{...}` (라인 ~1625)

### 6.2 파일명 패턴 변경
```javascript
// 라인 ~1735
const detectLang = n => {
  const t = n.toLowerCase();
  if (/_en[_.]/.test(t)) return 'en';
  if (/_ja[_.]/.test(t)) return 'ja';
  if (/_origin[_.]/.test(t) || /_kr[_.]/.test(t)) return 'kr';
  return null;
};
```
다른 패턴(예: `_KO_`, `_TRANS_EN_`)을 추가하려면 정규식 수정.

### 6.3 검수 대상 블록 클래스 변경
기본은 `.text-info`. 다른 마크업 템플릿을 쓴다면 bridge 내부의 query를 수정:
```javascript
// bridge 내부, 라인 ~1540
document.querySelectorAll('.text-info').forEach((b, i) => { ... });
// → document.querySelectorAll('.your-block-class')
```

### 6.4 한국어 정규식 확장
영어/일본어 외 다른 잔존 언어를 검출하려면:
```javascript
// bridge 내부, 라인 ~1390
const KO = /[가-힣]+/g;
// 예: 중국어 검출 추가
const ZH = /[一-鿿]+/g;
```

### 6.5 단축키 매핑
```javascript
// keydown 핸들러 라인 ~2650
if (e.key === '`') { /* KR 토글 */ }
if (e.key === 'F8') { /* 동기화 토글 */ }
if (e.ctrlKey && e.shiftKey && e.key === 'F') { /* 검색 패널 */ }
```

---

## 7. 파일명 규약 (Naming Convention)

### 7.1 기본 패턴
| 언어 | 패턴 | 예시 |
|---|---|---|
| EN | `*_en_*.html` 또는 `*_en.html` | `notice_en_544358.html` |
| JA | `*_ja_*.html` 또는 `*_ja.html` | `notice_ja_544359.html` |
| KR (원본) | `*_origin_*.html` 또는 `*_kr_*.html` | `notice_origin_984070.html` |

### 7.2 권장 작명 룰 (조직 도입 시)
```
{프로젝트}_{콘텐츠ID}_{언어}_{버전}.html
─────────────────────────────────────
oliveyoung_n2025q2_en_v3.html
```

→ 파일명에서 메타데이터를 추출할 수 있어 일괄 처리에 유리.

### 7.3 자동 분류 우선순위
1. 파일명 패턴 매칭 (`detectLang`)
2. 매칭 실패 → 사용자가 헤더 `📁 EN/JA/KR` 버튼으로 수동 선택

---

## 8. 배포 가이드

### 8.1 GitHub Pages (가장 간단)
```bash
git clone https://github.com/your-org/html-checker.git
cd html-checker
# 색상/단축키 등 커스터마이징
git add . && git commit -m "customize"
git push
# Settings → Pages → Source: main branch → Save
# 1~2분 후 https://your-org.github.io/html-checker/ 에서 접속
```

### 8.2 Nginx (사내 호스팅)
```nginx
server {
  listen 80;
  server_name qa.internal.example.com;
  root /var/www/html-checker;
  index index.html;

  # MIME 명시 (일부 환경)
  types { text/html html; }

  # 캐시 설정 (단일 파일이므로 적극적 캐싱)
  location = /index.html {
    add_header Cache-Control "public, max-age=3600, must-revalidate";
  }
}
```

### 8.3 S3 + CloudFront
```bash
# 버킷 생성 후
aws s3 cp index.html s3://my-html-checker/ --content-type "text/html; charset=utf-8"
# CloudFront 배포 설정 (Origin: S3, Default Root Object: index.html)
```

### 8.4 다중 환경 (개발/스테이징/운영)
단일 파일이므로 환경별로 같은 파일을 다른 도메인에 두면 끝.
```
qa-dev.flitto.com     ← 개발 (테스트용 데이터)
qa-staging.flitto.com ← 스테이징
qa.flitto.com         ← 운영
```

---

## 9. 브라우저 호환성 & 보안

### 9.1 호환성 매트릭스
| 브라우저 | 동작 | 비고 |
|---|---|---|
| Chrome 90+ | ✅ 완전 호환 | 권장 |
| Edge 90+ | ✅ 완전 호환 | Chromium 기반 |
| Firefox 90+ | 🟡 부분 호환 | iframe 렌더 차이로 검수 정확도 영향 가능 |
| Safari 15+ | 🟡 부분 호환 | postMessage / Blob URL 동작은 OK, 일부 CSS 차이 |
| IE 11 | ❌ 미지원 | ES6+ 사용 |

### 9.2 보안 고려사항
- **iframe sandbox**: 현재 `sandbox` 속성을 사용하지 않음 (Blob URL이라 same-origin policy 자체로 격리). 외부에서 받은 HTML을 신뢰할 수 없다면 `sandbox="allow-same-origin allow-scripts"` 추가 권장.
- **CSP**: 정적 호스팅이므로 별도 CSP 헤더 설정 가능. `script-src 'self' 'unsafe-inline'` 필요 (BRIDGE_SCRIPT는 인라인).
- **민감정보 처리**: 모든 처리는 브라우저 메모리에서만 발생, 서버로 업로드 안 됨. **단**, localStorage에 검색 히스토리가 저장됨 → 공용 PC에서는 정기 청소 필요.
- **HTTPS**: 권장. 단, file:// 로컬 실행도 동작 (PWA 동작은 제한).

---

## 10. 퍼포먼스 & 한계

### 10.1 측정된 퍼포먼스 (Chrome 120, MacBook Pro M1)
| 작업 | 시간 |
|---|---|
| HTML 파일 로드 (200KB) | ~50ms |
| iframe 렌더링 완료 | ~150ms (이미지 제외) |
| 다중 검색 (10색, .text-info 200개) | ~80ms |
| 한국어 자동 검출 | ~30ms |
| 미니맵 마커 200개 렌더링 | ~10ms |

### 10.2 알려진 한계
1. **`.text-info` 블록 수 1,000개 이상**: 검색 응답 시간 200ms 초과 가능 → 디바운스 권장
2. **이미지 다수 포함 HTML**: 스크롤 위치 동기화가 이미지 로드 완료 후에 정확해짐 (현재 `_emitScroll`을 3회 호출로 보완)
3. **메모리**: 3개 iframe 모두 활성 시 평균 200MB 사용. 매우 큰 HTML(>5MB)은 권장하지 않음
4. **모바일**: 반응형 미지원. 데스크톱 전용 (작업 도구 특성)

---

## 11. 확장 시나리오

### 11.1 추가 언어 지원 (예: 중국어)
- `detectLang` 정규식에 `_zh_` 추가
- `iframeScroll`, `filesLoaded`, `headerXBtn` 등 모든 lang 배열에 'zh' 추가
- 3분할 또는 4분할 패널 레이아웃 변경 (현재 2분할 + KR overlay)
- 영향 범위 약 30곳 (검색해서 'en', 'ja' 자리에 'zh' 추가)

### 11.2 검수 결과 서버 저장
현재는 클라이언트 메모리 only. 결과를 서버에 남기려면:
```javascript
// detectKorean() 끝부분에 추가
fetch('/api/qa-results', {
  method: 'POST',
  body: JSON.stringify({ filename, koreanFound: allRes.length, errors: S.errors })
});
```
백엔드 API 1개만 추가하면 됨.

### 11.3 자동화 (Headless Chrome)
Puppeteer로 자동 검수도 가능:
```javascript
const browser = await puppeteer.launch();
const page = await browser.newPage();
await page.goto('https://qa.example.com/');
// 파일 업로드 → 한국어 검출 결과 추출 → CSV로 저장
```
CI/CD 파이프라인에 통합 가능.

### 11.4 TM(Translation Memory) DB 연동
현재 TM은 사용자가 검색창에 직접 입력. DB 연동:
```javascript
// 사이드바에 "TM에서 불러오기" 버튼 추가
const tmEntries = await fetch('/api/tm?project=xxx').then(r => r.json());
tmEntries.forEach((entry, i) => {
  searchRows[i] = { en: entry.en, ja: entry.ja };
});
```

---

## 12. 트러블슈팅

### 12.1 흔한 문제와 해결책

#### Q: 파일을 드래그해도 인식이 안 됨
- **A**: 파일명에 `_en_`, `_ja_`, `_origin_` 패턴이 있는지 확인. 없으면 헤더의 `📁 EN/JA/KR` 버튼으로 수동 로드.

#### Q: 한국어 자동 검출이 누락됨
- **A**: HTML이 `.text-info` 클래스 기반 블록을 사용하는지 확인. 다른 마크업 템플릿을 쓴다면 [6.3 참조](#63-검수-대상-블록-클래스-변경).

#### Q: KR 원본이 백틱(`) 키로 안 뜸
- **A**:
  1. KR 파일(`_origin_*.html`)이 로드돼 있는지 확인 (헤더 라벨 확인)
  2. 검색 입력창에 포커스가 있는지 확인 (포커스 있으면 백틱이 입력으로 인식)
  3. 키보드 레이아웃 확인 (일부 한국어 키보드는 백틱 ≠ 한자 `₩`)

#### Q: 스크롤 동기화가 어긋남
- **A**: `📍 현재 위치 동기화` 버튼으로 1회 재정렬. 이미지 로드 완료 후에 다시 동기화하면 정확.

#### Q: 다크 모드에서 일부 텍스트가 안 보임
- **A**: 자체 다크 모드는 UI 컨테이너만 적용. iframe 내부 HTML은 원본 그대로 → 원본이 흰 배경에 흰 글자였다면 다크 모드와 무관하게 안 보일 수 있음.

#### Q: 검색 히스토리가 사라짐
- **A**: localStorage 기반. 시크릿 모드에서는 새로고침 시 초기화. 정상 모드에서도 브라우저 데이터 청소 시 삭제됨.

### 12.2 디버깅 팁
- DevTools 열고 콘솔에서:
  ```javascript
  // 현재 로드된 파일 상태
  console.log(filesLoaded);

  // 마지막 검색 결과
  console.log(_lastSearchResults);

  // iframe 스크롤 상태
  console.log(iframeScroll);
  ```
- iframe 내부 검사: DevTools → Elements → 좌측 iframe 선택 → `$0` 으로 컨텍스트 진입

---

## 13. 용어집

| 용어 | 정의 |
|---|---|
| **TM** | Translation Memory — 번역 메모리. 회사가 누적해온 번역 사전/용어집 |
| **글로서리(Glossary)** | 용어집. TM과 비슷한 개념이지만 보통 단어 단위 |
| **`.text-info`** | Flitto 마케팅 HTML 템플릿의 텍스트 블록 단위 클래스 |
| **bridge** | iframe 내부에 주입되는 검색/스크롤 처리 스크립트 |
| **minimap** | 패널 우측 14px 폭의 미니 스크롤바. 검색/한국어 마커가 표시되는 영역 |
| **unmatched** | 검색어가 EN/JA 중 한쪽에만 매칭된 줄 (좌측 ▶ 표시) |
| **KR overlay** | 백틱 키로 반대편 패널 위에 한국어 원본을 띄우는 모드 |
| **pill** | 검색 결과 카운트를 색상 박스로 표시한 UI 요소 (`#1 EN 5 / JA 3`) |
| **블록 인덱스** | 한 HTML 안에서 `.text-info` 요소의 순서. EN/JA 매칭 단위 |

---

## 부록 A. 의존성 다이어그램

```
[브라우저]
    ↓
[index.html] ← 단일 파일, 의존성 없음
    ↓ Blob URL
[iframe × 3] (EN / JA / KR)
    ↓ injected
[BRIDGE_SCRIPT] ← 검색·검출·스크롤 로직
    ↑ postMessage
[parent JS] ← UI 상태, 미니맵, 헤더
```

## 부록 B. 의사 결정 기록 (왜 이렇게 설계했나)

| 결정 | 이유 |
|---|---|
| **단일 파일** | 배포 단순화, CDN 캐시 효율, 의존성 0 |
| **iframe + Blob** | 원본 HTML의 CSS/JS 충돌 격리, Chrome 동일 렌더 보장 |
| **postMessage** | iframe 격리 + 비동기 안전 통신, 백엔드 불필요 |
| **localStorage** | 사용자별 설정/히스토리 유지, 서버 부담 없음 |
| **Vanilla JS** | 빌드 도구 불필요, 즉각 디버깅, 장기 유지보수 용이 |
| **한국어 정규식** | 라이브러리(예: franc) 의존성 회피, 빠른 검출 |

## 부록 C. 라이선스 & 저작권
- 사용 라이선스: 별도 [LICENSE](./LICENSE) 파일 참조 (All Rights Reserved by Blankers2)
- 사용 문의: 개발자 (GitHub: [@Blankers2](https://github.com/Blankers2)) 또는 Flitto 이미지팀

---

*이 문서는 v1.0 (2026-05) 기준입니다. 코드 변경 시 본 문서도 함께 갱신하세요.*
