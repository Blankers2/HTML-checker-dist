# 🧐 HTML Checker

> 마케팅 HTML 번역(EN/JA) 파일을 한국어 원본과 함께 나란히 비교·검수하는 브라우저 기반 QA 도구

🌐 **[바로 사용하기 → blankers2.github.io/HTML-checker-dist](https://blankers2.github.io/HTML-checker-dist/)**

별도 설치 없이 Chrome에서 바로 사용 가능합니다.

📑 **소개 보고서**: [pitch.html](https://blankers2.github.io/HTML-checker-dist/pitch.html) (비주얼 피치 문서)
📊 **발표용 PPT**: [HTML-Checker_기능_요구사항_정의.pptx](./HTML-Checker_기능_요구사항_정의.pptx) (전체 기능 · 요구사항 정의 — 구글 슬라이드 호환)
🛠 **개발자/도입 가이드**: [MIGRATION.md](./MIGRATION.md) (요구사항·아키텍처·커스터마이징 전체 정리)

---

## 🧰 전체 기능

| 기능 | 설명 |
|------|------|
| 📄 **2분할 비교 뷰** | EN / JA HTML을 좌우로 나란히 렌더링 (Chrome 동일 렌더) |
| 🇰🇷 **KR 원본 오버레이** | `` ` `` 토글 — 마우스 hover 반대편 패널이 KR로 즉시 전환 |
| 🔗 **스크롤 동기화** | 상시 동기화, 수동 위치 동기화, KR 단방향 동기 (모두 동시 작동) |
| 🇰🇷 **한국어 자동 검출** | 파일 로드 시 자동 실행, 헤더 배지로 즉시 결과 표시 |
| 🔎 **TM/글로서리 다중 검색** | 최대 10개 동시 — 줄바꿈·전각/반각·공백·대소문자·히라/카타 정규화 |
| 📜 **검색 히스토리** | localStorage 자동 저장, 한 번 클릭으로 복원 |
| 📝 **텍스트 추출** | `.text-info` 블록 기반 EN/JA 텍스트 일괄 추출 |
| 📋 **CSV 내보내기** | 한국어 검출 결과를 CSV로 다운로드 |
| 🔄 **새로고침** | 양쪽 iframe 즉시 재로드 (CDN 캐시 갱신용) |
| ↔️ **너비 조절 토글** | 좌우 비율 자유 조절 또는 1:1 고정 |
| 🌙 **다크/라이트 테마** | 원클릭 전환, 설정 보존 |

---

## 🚀 사용법

### 1. 파일 로드

EN · JA · KR 파일을 화면에 한꺼번에 드래그하면 파일명으로 자동 분류됩니다.

| 언어 | 파일명 패턴 |
|---|---|
| 🇺🇸 EN | `..._en_*.html` |
| 🇯🇵 JA | `..._ja_*.html` |
| 🇰🇷 KR (원본) | `..._origin_*.html` |

또는 헤더의 `📁 EN` / `📁 JA` / `📁 KR` 버튼으로 개별 선택.

### 2. KR 원본 비교

1. KR 파일(`_origin_*.html`)을 함께 로드
2. EN/JA 패널 위에서 `` ` `` 키 → KR 오버레이 ON
3. 마우스를 다른 패널로 옮기면 KR이 반대편으로 자동 전환
4. 다시 `` ` `` → OFF

### 3. 다중 검색

1. 헤더 `🔎 검색` → 사이드바 열기
2. `＋ 추가`로 검색 행 최대 10개 추가
3. 결과 카운트 pill 클릭 → 해당 검색어 결과만 필터링

---

## ⌨️ 단축키

| 키 | 동작 |
|---|---|
| `` ` `` | KR 원본 비교 ON/OFF |
| `Esc` | KR 해제 → 검색어 비우기 → 사이드바 토글 |
| `F8` | 상시 동기화 ON/OFF |
| `Shift` + `Wheel` | 양쪽 동기 스크롤 |
| `Ctrl` + `Shift` + `F` | 검색 패널 열기 |
| `Ctrl` + `Shift` + `X` | 검색 결과 초기화 |

---

## 🎨 다중 검색 색상 팔레트 (10색)

| # | 색상 | hex | # | 색상 | hex |
|---|---|---|---|---|---|
| 1 | 노랑 | `#ffaa00` | 6 | 핑크 | `#ec4899` |
| 2 | 파랑 | `#3b82f6` | 7 | 청록 | `#06b6d4` |
| 3 | 초록 | `#10b981` | 8 | 라임 | `#84cc16` |
| 4 | 보라 | `#a855f7` | 9 | 빨강 | `#ef4444` |
| 5 | 주황 | `#f97316` | 10 | 검정 | `#1f2937` |

---

## 🛠 기술 스택

| 항목 | 기술 |
|------|------|
| 프론트엔드 | Vanilla HTML/CSS/JS (단일 파일, 약 2,400줄) |
| 렌더링 | Blob URL + iframe (Chrome 동일 렌더 보장) |
| 통신 | postMessage Bridge (cross-origin 안전) |
| 스크롤 동기 | requestAnimationFrame 기반 저지연 동기화 |
| 영속 저장 | localStorage (검색 히스토리, 사이드바 상태, 테마) |
| 호스팅 | GitHub Pages (정적 배포) |

---

## 🏛 아키텍처

```
index.html (Parent)
├── Header
│   ├── Left:   HTML Checker | 📁 EN | 📁 JA | 📁 KR
│   ├── Center: 동기화 | 📍 위치 동기화 | 🔄 새로고침 | 🔎 검색 | 🇰🇷 한국어 ▾
│   └── Right:  ⇔ 너비조절 | 🇰🇷 원본 | 📝 추출 ▾ | 📋 내보내기 | ⌨️ 단축키 ▾ | 🌙
├── Sidebar (TM 다중 검색 전용)
│   └── 🔎 + 📜 히스토리 — 검색 행 최대 10개 + 옵션 + 결과
└── Split View
    ├── EN Panel  → iframe + minimap (검색 마커 + 스크롤바)
    ├── JA Panel  → iframe + minimap
    └── KR overlay → 사전 렌더 hidden iframe, ` 토글 시 반대편에 absolute 위치
```

**BRIDGE_SCRIPT**: 모든 iframe(EN/JA/KR)에 주입되어 스크롤·검색·하이라이팅·드래그를 `postMessage`로 parent와 양방향 통신.

---

---

## 📄 License

Copyright (c) 2025 [Blankers2](https://github.com/Blankers2). All rights reserved.

사전 서면 허가 없이 복사·수정·배포·사용을 금지합니다.
자세한 내용은 [LICENSE](./LICENSE) 파일을 참고하세요.

<img src="icon.png" width="20" align="top"> Built for Flitto Inc.
