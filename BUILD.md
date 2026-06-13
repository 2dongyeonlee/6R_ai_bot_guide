# BUILD — 슬림 버전 구축 가이드

이 가이드는 별도의 빌드 도구(번들러, 프레임워크) 없이 동작하는
**단일 HTML 슬림 버전**입니다. 의존성 설치 없이 브라우저만 있으면 됩니다.

## 1. 요구 사항

- 최신 웹 브라우저 (Chrome / Edge / Safari / Firefox)
- (선택) 로컬 정적 서버 — Python 3 또는 Node.js
- 인터넷 연결 — 폰트(Google Fonts)를 외부에서 불러옵니다

> 빌드 산출물 = `index.html` 단 하나. CSS와 JS가 모두 인라인으로 포함되어 있어
> 파일 하나만 배포하면 그대로 동작합니다.

## 2. 로컬에서 보기

### 방법 A — 파일 직접 열기 (가장 간단)

`index.html`을 브라우저로 드래그하거나 더블클릭합니다.

```bash
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows
```

### 방법 B — 로컬 서버 (권장)

일부 브라우저는 `file://`에서 동작이 제한될 수 있으므로 정적 서버 사용을 권장합니다.

```bash
# Python 3
python3 -m http.server 8000

# 또는 Node.js
npx serve .
```

브라우저에서 <http://localhost:8000> 접속.

## 3. 수정하기

가이드 내용·스타일·스크립트는 모두 `index.html` 한 파일 안에 있습니다.

| 영역 | 위치 |
|------|------|
| 디자인 토큰 (색상·여백) | `<style>` 상단의 `:root { --bg, --accent ... }` |
| 사이드바 / 목차 | `.sidebar`, `.nav-item` 마크업 |
| 본문 단계 (Step 1~6) | 각 `section` (`overview`, `s1`~`s6`, `connect`, `code` 등) |
| 진행 상태 / 네비게이션 동작 | 하단 `<script>`의 `go(...)` 함수 |

수정 후 브라우저를 새로고침하면 바로 반영됩니다. 컴파일 단계가 없습니다.

## 4. 배포

단일 파일이므로 어떤 정적 호스팅에도 그대로 올릴 수 있습니다.

### GitHub Pages

1. 저장소 **Settings → Pages**
2. Source: `Deploy from a branch`
3. Branch: 배포할 브랜치 / 루트(`/`) 선택 후 저장
4. 잠시 후 `https://<사용자>.github.io/6r_ai_bot_guide/` 에서 확인

### Cloudflare Pages

1. <https://dash.cloudflare.com> → **Workers & Pages → Create → Pages**
2. 이 저장소 연결
3. **Build command** 비워둠 · **Build output directory** `/` (루트)
4. 배포 완료 후 제공되는 `*.pages.dev` 도메인으로 접속

### 기타

`index.html`을 그대로 복사해 사내 위키, 공유 드라이브, Netlify, Vercel 등
어디든 올릴 수 있습니다.

## 5. 체크리스트

- [ ] `index.html`이 단일 파일로 정상 렌더되는지 확인
- [ ] 사이드바 네비게이션(Step 이동)이 동작하는지 확인
- [ ] 외부 링크(BotFather, Dify, Cloudflare 등)가 열리는지 확인
- [ ] 배포 URL에서 모바일/PC 양쪽 레이아웃 확인
