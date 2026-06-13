# 6R AI 봇 가이드

> 텔레그램 기반 AI 비서봇을 **Dify 없이 Cloudflare + Claude만으로** 구현하는 가이드.
> 사내 AI 전환 프로젝트 (조직명 비공개).

---

## 이게 뭔가요

각 구성원이 텔레그램에서 자기 전담 AI 비서봇과 대화하는 시스템입니다.

봇이 하는 일:
- 단체방 메시지·보고·파일을 자동으로 읽고 저장
- 올라온 PDF·이미지를 자동 분석·요약
- "OO 자료 보내줘", "6/15 보고 뭐 있어?" → 쌓인 자료에서 검색
- 매일 아침 09:00 주요 보고·일정·의사결정 자동 브리핑
- 실시간 외부 뉴스·정보 검색 (Tavily)
- 구성원 간 메시지 전달

---

## 구조

```
[텔레그램] → [Cloudflare Workers] → [D1 / KV] → [Claude API] → [텔레그램]
                                              → [Tavily] (외부 검색)
```

- **Cloudflare Workers**: 봇 로직 (서버리스, 안 쓰면 0원)
- **D1**: 모든 메시지·요약·파일 정보 저장
- **KV**: 시스템 프롬프트·대화 히스토리
- **Claude**: 요약·분석·답변 (두뇌)
- **Dify 제외**: 중간 도구 없이 Worker가 Claude 직접 호출 → 빠르고, 기밀이 외부 경유 안 함

---

## 비용 (월 결제액)

| 항목 | 비용 |
|------|------|
| Cloudflare | 무료 (10GB) |
| Claude | $20~50 |
| Tavily | 월 1,000건 무료 |
| **합계** | **약 $20~80 (3~10만원)** |
| 초기 구축 | **0원** |

---

## 문서

- **[BUILD.md](./BUILD.md)** — 봇을 처음부터 만드는 전체 가이드 (새로 구현하려는 사람용)
- **[CHANGELOG.md](./CHANGELOG.md)** — 개발 이력, 해결한 버그, 다음 단계
- **index.html** — GitHub Pages용 가이드 페이지

---

## 빠른 시작

```bash
# 1. 도구 설치
npm install -g wrangler && wrangler login

# 2. DB·KV 생성
wrangler d1 create 6r-ai-db
wrangler kv namespace create PROMPT

# 3. 비밀키 등록
wrangler secret put TELEGRAM_BOT_TOKEN
wrangler secret put ANTHROPIC_API_KEY

# 4. 배포
wrangler deploy

# 5. 웹훅 연결
curl "https://api.telegram.org/bot<토큰>/setWebhook?url=<WorkerURL>"
```

자세한 단계는 [BUILD.md](./BUILD.md) 참고.
