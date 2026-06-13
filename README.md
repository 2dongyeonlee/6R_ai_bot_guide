# 6R AI 봇 가이드

6R전략 담당 구성원을 위한 **AI 비서봇 생성 가이드**입니다.
텔레그램 봇부터 LLM API 발급, Dify 에이전트 구성, Cloudflare 배포, GitHub 저장,
테스트까지 6단계로 따라 할 수 있도록 정리한 단일 HTML 문서입니다.

- 제작: 이동연 TL · AI Comm 기획팀
- 문의: 텔레그램 [@dylee_ai_bot](https://t.me/dylee_ai_bot) 또는 이동연 TL에게 직접

## 구성

이 저장소는 **별도 빌드 과정이 필요 없는 단일 파일 문서**입니다.

| 파일 | 설명 |
|------|------|
| `index.html` | 가이드 본문 (CSS/JS 인라인 포함, 단일 파일로 동작) |
| `README.md` | 저장소 개요 (이 문서) |
| `BUILD.md` | 슬림 버전 구축 · 로컬 실행 · 배포 가이드 |
| `CHANGELOG.md` | 변경 이력 |

## 빠른 시작

```bash
# 저장소 클론 후
open index.html        # macOS
# 또는 브라우저로 index.html 파일을 직접 열기
```

로컬 서버로 보고 싶다면:

```bash
python3 -m http.server 8000
# 브라우저에서 http://localhost:8000 접속
```

자세한 구축 · 배포 방법은 [BUILD.md](./BUILD.md)를 참고하세요.

## 가이드 목차

1. **Step 1.** 텔레그램 봇 생성 (BotFather)
2. **Step 2.** LLM API 발급 (Anthropic Claude / OpenAI / Google AI Studio)
3. **Step 3.** Dify 에이전트 구성
4. **Step 4.** Cloudflare 배포
5. **Step 5.** GitHub 저장
6. **Step 6.** 테스트 & 완료

심화: 봇끼리 연결하기 · 예시 코드 · AI 코드 생성 프롬프트 · 시스템 프롬프트 예시 · FAQ

## 주요 외부 서비스

- 텔레그램 BotFather — <https://t.me/BotFather>
- Anthropic Console — <https://console.anthropic.com>
- OpenAI Platform — <https://platform.openai.com>
- Google AI Studio — <https://aistudio.google.com>
- Dify — <https://cloud.dify.ai>
- Cloudflare — <https://dash.cloudflare.com>
- GitHub — <https://github.com>
