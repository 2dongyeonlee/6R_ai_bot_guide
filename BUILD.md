# 6R AI 비서봇 구축 가이드 (슬림 버전)

> Dify 없이 Cloudflare + Claude만으로 구현하는 텔레그램 AI 비서봇.
> 이 문서대로 따라하면 새 봇 하나를 처음부터 끝까지 만들 수 있습니다.

---

## 1. 전체 구조 한눈에

```
[텔레그램]
    ↓ 메시지·파일 수신
[Cloudflare Workers]   ← 봇 로직 (worker.js)
    ↓
[Cloudflare D1]        ← 모든 메시지·요약·파일 정보 저장
[Cloudflare KV]        ← 시스템 프롬프트·대화 히스토리
    ↓
[Claude API]           ← 요약·분석·답변 (두뇌)
[Tavily API]           ← 실시간 외부 검색 (선택)
    ↓
[텔레그램]             ← 답변 전송
```

**핵심 설계 원칙**
- 들어온 모든 것(텍스트·파일·이미지)을 텍스트화해서 D1에 저장한다.
- 저장된 자료를 브리핑·검색·답변이 재사용한다.
- Dify 같은 중간 도구 없이 Worker가 Claude를 직접 호출한다. (빠르고, 기밀이 외부 경유 안 함)

---

## 2. 준비물

| 항목 | 용도 | 비용 |
|------|------|------|
| 텔레그램 계정 | 봇 생성 (BotFather) | 무료 |
| Cloudflare 계정 | 서버·DB·KV | 무료 (10GB까지) |
| Anthropic API 키 | Claude 두뇌 | 사용량 비례 ($20~50/월) |
| Tavily API 키 | 외부 검색 (선택) | 월 1,000건 무료 |
| Node.js + wrangler | 배포 도구 | 무료 |

---

## 3. 단계별 구축

### 3-1. 텔레그램 봇 생성

1. 텔레그램에서 `@BotFather` 검색 → 대화 시작
2. `/newbot` 입력
3. 봇 이름, 사용자명(`_bot`으로 끝나야 함) 설정
4. 발급된 **봇 토큰** 저장 (예: `1234567:ABC-xyz...`)
5. `/setprivacy` → 해당 봇 선택 → `Disable`
   (그룹방에서 모든 메시지를 받으려면 필수)

### 3-2. Cloudflare 세팅

```bash
# wrangler 설치
npm install -g wrangler

# 로그인
wrangler login
```

### 3-3. D1 데이터베이스 생성

```bash
wrangler d1 create 6r-ai-db
```

출력되는 `database_id`를 복사해둡니다.

**messages 테이블 생성** (아래 스키마를 schema.sql로 저장 후 실행):

```sql
CREATE TABLE IF NOT EXISTS messages (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  telegram_message_id TEXT,
  room_id TEXT NOT NULL,
  room_title TEXT,
  sender_id TEXT NOT NULL,
  sender_name TEXT,
  content TEXT NOT NULL,
  status_tag TEXT DEFAULT '',
  field_tag TEXT DEFAULT '',
  milestone_date TEXT DEFAULT '',
  file_id TEXT DEFAULT '',
  file_name TEXT DEFAULT '',
  summary TEXT DEFAULT '',
  action_items TEXT DEFAULT '',
  needs_escalation INTEGER DEFAULT 0,
  info_tag TEXT DEFAULT '',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

```bash
wrangler d1 execute 6r-ai-db --remote --file=schema.sql
```

### 3-4. KV 네임스페이스 생성

```bash
wrangler kv namespace create PROMPT
```

출력되는 `id`를 복사해둡니다. (시스템 프롬프트·대화 히스토리 저장용)

### 3-5. wrangler.toml 작성

```toml
name = "내봇이름"
main = "worker.js"
compatibility_date = "2024-11-01"
compatibility_flags = ["nodejs_compat"]
keep_vars = true

[vars]
BRIEFING_CHAT_ID = "-방ID1,-방ID2"   # 아침 브리핑 보낼 방
BOT_USERNAME = "내봇이름_bot"          # @ 제외

[triggers]
crons = ["0 0 * * 1-5"]   # UTC 00:00 = KST 09:00, 월~금

[[d1_databases]]
binding = "DB"
database_name = "6r-ai-db"
database_id = "위에서_복사한_DB_ID"

[[kv_namespaces]]
binding = "PROMPT"
id = "위에서_복사한_KV_ID"
preview_id = "위에서_복사한_KV_ID"
```

### 3-6. 비밀 키 등록

```bash
wrangler secret put TELEGRAM_BOT_TOKEN   # BotFather 토큰
wrangler secret put ANTHROPIC_API_KEY    # Claude API 키
wrangler secret put TAVILY_API_KEY       # (선택) 외부 검색
```

### 3-7. worker.js 배포

worker.js를 같은 폴더에 두고:

```bash
wrangler deploy
```

배포 후 출력되는 Worker URL을 복사 (예: `https://내봇이름.xxx.workers.dev`)

### 3-8. 텔레그램 웹훅 연결

봇 토큰과 Worker URL을 넣어 실행:

```bash
curl "https://api.telegram.org/bot<봇토큰>/setWebhook?url=<WorkerURL>"
```

`{"ok":true}` 나오면 연결 완료. 봇에게 메시지를 보내면 응답합니다.

---

## 4. 핵심 코드 구조 (worker.js)

### 진입점

```javascript
export default {
  async fetch(request, env) {
    // 텔레그램이 보낸 메시지 수신 → handleMessage
  },
  async scheduled(_event, env, ctx) {
    // cron(매일 09:00) → 아침 브리핑
    ctx.waitUntil(runMorningBriefing(env));
  },
};
```

### 메시지 라우팅 (handleMessage)

처리 순서가 중요합니다. **저장이 먼저, 라우팅이 나중**입니다.

```
1. 봇 자신 메시지면 무시
2. /설정 명령이면 → 시스템 프롬프트 수정
3. 파일·이미지면 → 추출·저장·요약 (ingestAndSummarize)
4. 텍스트면 → 잡담 아니면 먼저 DB 저장 (saveMessage)
5. #보고 등 태그 있으면 → 업무보고 처리 (handleReport)
6. 봇 멘션·질문이면 → 답변 생성 (handleQuery)
7. 그 외는 저장만 (응답 없음)
```

> 과거에 "저장이 라우팅 뒤에 있어서 메시지가 누락"되는 버그가 있었음.
> 반드시 저장을 먼저 하도록 유지할 것.

### 질문 처리 (handleQuery)

```
1. 대화 히스토리 로드 (KV, 직전 1~2턴)
2. 내부 자료 검색 (searchMemory → D1)
3. 내부에 없으면 외부 검색 (searchWeb → Tavily)
4. 질문 복잡도 판단 → 모델 선택
   - 단순: Haiku (claude-haiku-4-5)
   - 분석·브리핑: Sonnet (claude-sonnet-4-6)
5. Claude 호출 → 답변 생성
6. 히스토리 저장 (KV)
7. 파일 요청이면 해당 파일 재전송
```

### 모델 라우팅 (비용 절감 핵심)

```javascript
const MODEL_FAST  = "claude-haiku-4-5-20251001";  // 단순 대화 (저렴)
const MODEL_SMART = "claude-sonnet-4-6";          // 분석·브리핑

function isComplexQuery(query, hasFile) {
  if (hasFile) return true;
  if (query.length > 100) return true;
  return /(요약|분석|보고|브리핑|정리|검토|작성|전략|방향|판단|비교|예측)/.test(query);
}
```

---

## 5. 데이터 저장 방식

모든 메시지는 `insertMessage`로 D1에 저장됩니다. 핵심 컬럼:

| 컬럼 | 내용 |
|------|------|
| room_id / room_title | 어느 방에서 왔는지 |
| sender_id / sender_name | 누가 보냈는지 |
| content | 원문 (또는 파일 추출 텍스트) |
| status_tag | #보고 #공유 #Fup #일정 |
| field_tag | #AI #월간보고 등 분야 |
| milestone_date | 마감일 (YYYY-MM-DD) |
| file_id / file_name | 텔레그램 파일 참조 |
| summary | 파일·문서 요약 (검색용 키워드 포함) |

> **저장 시 주의**: sender_id가 NULL이면 INSERT가 조용히 실패함.
> 모든 값을 `String()`으로 감싸고, insertMessage 전체를 try-catch로 감쌀 것.

---

## 6. 파일 처리 (ingestAndSummarize)

| 형식 | 처리 방식 | 비고 |
|------|----------|------|
| PDF | Claude에 base64 document로 직접 전달 | 32MB 이하, 표·레이아웃까지 인식 |
| 이미지 (jpg/png/webp/gif) | Claude Vision으로 텍스트화 (describeImage) | |
| PPTX/DOCX/XLSX | 파일에 Reply 후 요약 요청 시 텍스트 추출해 답변 | Reply 방식 |

파일이 올라오면 `Promise.all`로 두 가지를 **동시에** 생성 (속도):
1. **사용자 답변용** (Sonnet) — 일정 / 의사결정사항 / 핵심요약 양식
2. **DB 저장용 JSON** (Sonnet) — summary(검색 키워드 포함) + action_items + needs_escalation

> PDF 추출은 별도 라이브러리 없이 Claude API에 직접 base64로 보냄.
> Cloudflare Worker에서 `btoa`로 인코딩 → Anthropic `document` 타입으로 전달.
> 이래야 표·서식이 살아있는 채로 분석됨.

---

## 6-1. 자료 검색 (searchMemory) — 하이브리드

봇에게 "OO 자료 공유해줘", "6/15 보고 뭐 있어?" 하면 D1에서 검색.

**작동 방식 (키워드 + 의미 동시)**
1. 키워드 검색(likeSearch) — content·summary·file_name·sender_name LIKE
2. 의미 검색(Vectorize) — 질문을 임베딩해 의미 유사 항목 조회
3. 두 결과를 병합·중복제거 → 상위 12개 반환

**발신자 명시 검색 (searchBySender)**
- "예슬TL이 보고한 자료"처럼 사람이 명시되면 해당 인원 메시지 우선 필터
- NAME_ALIASES로 이름·성·영문명 변형 자동 매칭

**자료 = 파일 + 사진 + 텍스트 통합**
- 단체방 파일 → copyMessage로 원본 전송
- 동일 파일이 단체방·DM 양쪽에 있으면 단체방 버전 우선
- DM 전용 파일 → 저장된 전체 내용을 텍스트로 제공
- 텍스트 보고 → "누가·무슨 내용" 형태로 요약 답변

## 6-2. 외부 검색 (searchWeb / Tavily)

내부 자료에 없거나 "검색해줘/찾아봐줘" 요청이면 Tavily 호출.

```javascript
// hits.length < 3 이거나 "검색/뉴스/최신" 키워드면 외부 검색
const res = await fetch("https://api.tavily.com/search", {
  method: "POST",
  body: JSON.stringify({
    api_key: env.TAVILY_API_KEY,
    query, search_depth: "basic", max_results: 5, include_answer: true,
  }),
});
```

답변에 결과 반영 + 출처 URL 2개만 제공.

## 6-3. 메시지 전달 (parseForwardRequest)

"OO에게 전해줘 / OO님께 전달해줘" → D1에서 sender_name으로 chat_id 조회 → 해당 사람 DM으로 전송.

> DB에 chat_id가 있는 사람만 가능 (= 그 사람이 봇과 한 번이라도 대화했어야 함).
> 없으면 "직접 연락 부탁드립니다"로 안내.

---

## 7. 아침 브리핑

cron(`0 0 * * 1-5` = KST 09:00 월~금)에 자동 실행.

D1에서 status_tag별로 분류 → Claude가 양식대로 정리 → BRIEFING_CHAT_ID 방들에 전송.

**양식**
```
📅 YYYY-MM-DD Daily Briefing
📌 주요 일정   (임원·담당급 언급, 날짜 확정 일정)
🔔 보고        (팀장·TL 진행 보고)
💡 의사결정 필요
📋 주요 내용   (방별 핵심 공유)
```
분류 기준:
- 임원(담당급 이상)·CEO·총괄 관련 → 📌 주요 일정
- 팀장·TL 진행 보고 → 🔔 보고
- 판단·승인 필요 → 💡 의사결정
- 오늘보다 과거 날짜 항목은 자동 제외 (코드 필터)

> 날짜·태그 분류는 코드(SQL)가 결정적으로 처리하고, 문장만 Claude가 생성.
> 이래야 날짜 계산이 틀리지 않음.

---

## 8. 시스템 프롬프트 규칙 (검증된 것)

봇 응답 품질을 좌우하는 핵심. KV에 저장되어 `/설정` 명령으로 재배포 없이 수정 가능.

```
- 존댓말, 짧고 간결하게, 바로 본론
- 마크다운(**,#,*) 금지. HTML <b>태그만 사용
- 문장 끝 "~입니다" 금지. 명사형으로 끝낼 것
- 블릿포인트(•) 사용 (텔레그램 모바일 가독성)
- 긴급(오늘·D-1)은 🚨, D-7 이내는 ⚠️ 표시
- 추측 금지. 없는 정보는 만들지 않음
- 짧은 감탄사·불만엔 짧게 받아침
- Tavily 외부 검색 연동됨. "검색 불가"라고 답하지 말 것
```

---

## 9. 팀원 봇 복제 시 바꿀 것

기존 봇을 복제해 새 팀원 봇을 만들 때 변경할 항목:

| 항목 | 위치 |
|------|------|
| 봇 토큰 | `wrangler secret put TELEGRAM_BOT_TOKEN` |
| 봇 이름 | wrangler.toml `name`, `BOT_USERNAME` |
| 브리핑 방 | wrangler.toml `BRIEFING_CHAT_ID` |
| 시스템 프롬프트 | DEFAULT_SYSTEM_PROMPT (담당자 이름·역할) |
| Worker URL 웹훅 | setWebhook 다시 실행 |

> D1·KV는 봇마다 별도로 만들 것. (database_id 공유 금지)

---

## 10. 자주 막히는 곳 (트러블슈팅)

| 증상 | 원인 | 해결 |
|------|------|------|
| 메시지 저장 안 됨 | sender_id NULL로 INSERT 실패 | String() 캐스팅 + try-catch |
| 그룹방에서 응답 없음 | BotFather privacy 설정 | `/setprivacy` → Disable |
| 외부 검색 안 됨 | TAVILY_API_KEY 미등록 | secret 등록 확인 |
| 브리핑 안 옴 | cron 시간(UTC 기준) | `0 0 * * 1-5` = KST 09:00 |
| 한글 깨짐 | PowerShell 직접 수정 | 코드 수정은 에디터/Codex로, UTF-8 유지 |

---

## 11. 비용 (월 결제액 기준)

| 항목 | 비용 |
|------|------|
| Cloudflare (서버·저장) | 무료 (10GB, 초과 100GB당 약 2천원) |
| Claude (두뇌) | $20~50 (사용량 비례) |
| Tavily (검색) | 월 1,000건 무료, 초과 시 $30~ |
| **기본 운영 합계** | **약 $20~80 (월 3~10만원)** |
| 초기 구축 | **0원** (전담 개발자 없이 내부 구현) |

---

## 12. 향후 고도화 (검토 중)

| 단계 | 기능 | 추가 비용 |
|------|------|----------|
| RAG (적용 중) | 키워드+의미 하이브리드 검색 가동. 색인 확대 중 | Cloudflare Vectorize |
| 구글 연동 | 드라이브·캘린더 자료 직접 읽기 | API 무료 (두뇌 비용만) |
| 학습형 에이전트 | 보고·응답 학습 → 선제 제공 (Hermes 등) | 서버 $10~20 + 두뇌 별도 |

> 1차는 안정성 위해 검증된 Claude·Cloudflare로 구현.
> 쌓인 자료를 기억하고 선제 제시하는 방향으로 디벨롭 검토.
> AI·검색 도구는 빠르게 발전 중이라 더 낫고 저렴한 옵션으로 변경될 수 있음.
