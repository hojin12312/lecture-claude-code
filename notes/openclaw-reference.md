# OpenClaw 참조

> 웹 조사 결과 요약 (2026-04-25)

## 개요

OpenClaw는 오스트리아 개발자 Peter Steinberger가 2025년 11월에 공개한 **오픈소스 자율 AI 에이전트**입니다.
LLM을 실제 소프트웨어에 연결해 메시징 플랫폼(텔레그램 등)을 UI로 사용합니다.

- **GitHub (org)**: `github.com/openclaw/openclaw` (개발자 개인 GitHub은 `github.com/steipete`)
- **별 수**: 363K+ (2026-04-25 시점). 시계열: 145K (2026-02) → 247K (2026-03 초) → 280K+ (2026-03 후반) → 337K (2026-04 초) → 363K (2026-04-25)
- **라이선스**: MIT

## 작동 방식

- LLM + 100개 이상의 내장 스킬(이메일 읽기, 쉘 커맨드 실행, 웹 브라우징, API 제어 등)
- 메시징 플랫폼을 UI로 사용 (텔레그램, Discord 등)
- 로컬 실행 가능 (로컬 LLM 연결 지원)

## 이름 변천

1. **Clawdbot** (2025-11 출시)
2. **Moltbot** (2026-01-27 → Anthropic 상표 이슈로 변경)
3. **OpenClaw** (2026-01-30 ~ 현재)

## 주요 특징

- 단순 챗봇이 아닌 **자율 실행**: 멀티스텝 태스크를 스스로 처리
- **Ollama v0.20.5 (2026-04-09)부터 통합 지원** — WhatsApp/Telegram/Discord 채널 설정. v0.21.0은 단순 버그픽스 (`--yes` 옵션 등)
- **개발자 Peter Steinberger OpenAI 합류 (2026-02-14)** — TechCrunch / Fortune / Wikipedia(OpenClaw + Peter Steinberger 두 페이지 모두) / Sam Altman 트윗 (status 2023150230905159801) 다중 1차 출처 확인. **OpenClaw는 재단으로 이전, 오픈·독립 유지** (OpenAI 후원). 출처: en.wikipedia.org/wiki/OpenClaw, steipete.me/posts/2026/openclaw

## 보안 고려사항

- 이메일·캘린더·API 접근 권한을 가지므로 설정 오류 시 데이터 유출 위험
- **프롬프트 인젝션 취약점**: 처리하는 데이터 안에 악성 지시가 포함되면 LLM이 그대로 따를 수 있음

## 강의 연관성

- README.md 섹션 5 (에이전트 네트워크)의 실제 사례로 언급 가능
- "LLM을 에이전트 오케스트레이터로 쓰는 실용적 예시" 포지션
- Claude Code와 비교: CC는 개발 특화 에이전트, OpenClaw는 범용 태스크 에이전트

## 추천 모델 (로컬 실행 시, 2026-04)

커뮤니티 투표 기준 상위:
1. Qwen3 14B / 32B
2. Llama 3.3 70B (Q4)
3. DeepSeek 소형 MoE 계열
