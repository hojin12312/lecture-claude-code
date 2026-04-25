# Claude Code 명령어 참조 (로컬 전용, gitignore)

강의 자료 작성 시 참조용. 웹 서치 없이 확인할 수 있도록 정리.
마지막 검증: 2026-04-25 (공식 changelog 2.1.118 반영)

---

## 빌트인 슬래시 커맨드

| 명령어 | 설명 | 확인 |
|---|---|---|
| `/help` | 사용 가능한 명령어 목록 표시 | ✅ |
| `/clear` | 대화 히스토리 초기화 (컨텍스트 리셋) | ✅ |
| `/compact [지침]` | 컨텍스트를 요약 압축 (선택적으로 집중할 내용 지정 가능) | ✅ |
| `/usage` | 현재 세션 토큰 사용량·비용·통계 (v2.1.118에서 `/cost` + `/stats` 통합) | ✅ |
| `/cost` | `/usage`로 통합됨 (backward compat alias) | 🟡 deprecated |
| `/stats` | `/usage`로 통합됨 | 🟡 deprecated |
| `/status` | 계정·시스템 상태 확인 | ✅ |
| `/model` | 사용 모델 변경 | ✅ |
| `/effort` | Opus 4.7 reasoning effort 조절 (`xhigh` 포함). `--effort` 플래그와 동등 | ✅ (v2.1.118 신규 UI) |
| `/theme` | 커스텀 테마 적용·관리 (v2.1.118 신규) | ✅ |
| `/fast` | Fast 모드 토글 (Opus 4.6 기반, 출력 속도 향상) | ✅ |
| `/plan` | Plan 모드 토글 — 파일 수정 없이 계획만 제안 | ✅ |
| `/memory` | 메모리 파일 편집 | ✅ |
| `/init` | 현재 프로젝트에 CLAUDE.md 자동 생성 | ✅ |
| `/btw` | 메인 흐름을 건드리지 않고 곁가지 질문 처리 | ✅ (호진님 확인) |
| `/branch` | 현재 컨텍스트를 공유하면서 다른 방향 탐색 | ✅ (호진님 확인) |

---

## 스킬 기반 슬래시 커맨드

스킬은 사용자 또는 Anthropic이 정의한 확장 명령어. `/스킬명` 형태로 호출.

| 명령어 | 설명 |
|---|---|
| `/init` | CLAUDE.md 초기화 (빌트인과 동일) |
| `/review` | 현재 브랜치 PR 검토 |
| `/security-review` | 보안 검토 |
| `/simplify` | 변경된 코드 품질·간결성 검토 및 수정 |
| `/loop [간격] [명령어]` | 명령어 반복 실행 (예: `/loop 5m /review`) |
| `/schedule` | 예약 원격 에이전트 설정 |
| `/update-config` | settings.json 변경 (hooks, 권한 설정 등) |
| `/keybindings-help` | 키보드 단축키 커스터마이징 |
| `/fewer-permission-prompts` | 자주 쓰는 명령어 허용 목록 자동 등록 |
| `/claude-api` | Claude API / Anthropic SDK 관련 작업 |

---

## 키보드 단축키

| 단축키 | 동작 |
|---|---|
| `Shift + Tab` | Plan 모드 토글 |
| `Esc` | 현재 작업 중단 |
| `↑` | 이전 명령어 불러오기 |
| `v` | **Vim visual mode 진입** (v2.1.118 신규) — 선택·연산자·visual feedback 지원 |
| `V` | **Vim visual-line mode 진입** (v2.1.118 신규) — 라인 단위 선택 |

---

## 특수 입력 방식

| 형태 | 동작 |
|---|---|
| `@파일명` | 파일을 컨텍스트에 직접 주입 (예: `@CONTINUE.md`) |
| `! 명령어` | 셸 명령어를 세션에서 직접 실행 후 결과를 대화에 포함 |
| `#태그` | 메모리 태그 (사용 방식은 별도 확인 필요) |

---

## 강의 자료 내 언급 현황

- README.md 섹션 6: `/btw`, `/branch`, `/plan` 설명
- README.md 섹션 7: `Shift+Tab`, `/plan` 설명
- appendix/claude-code.md: CLAUDE.md·Hooks·Skills·MCP 상세

---

## 미확인 / 업데이트 필요 항목

- `#태그` 입력 방식 정확한 동작 미확인
- 스킬 목록은 설치 환경마다 다를 수 있음 (커스텀 스킬 혼재)
- `/compact` 옵션 파라미터 정확한 문법 재확인 권장

---

## v2.1.118 (2026-04-23) 기타 주목할 변경

- **Hooks에서 MCP 도구 직접 호출** 가능
- `DISABLE_UPDATES` 환경 변수 — 업데이트 경로 전면 차단
- WSL(Windows) 세션이 Windows 쪽 관리형 설정 상속
- Linux: subprocess sandboxing (PID namespace isolation)
- Cross-user prompt caching 개선
- Interactive Google Vertex AI · Bedrock setup wizard
- `Monitor` 도구 — 백그라운드 이벤트 스트리밍
- `CLAUDE_CODE_PERFORCE_MODE` 환경 변수
- Write 도구 diff 속도 +60% (tabs 포함 파일)

> 공식 문서: https://code.claude.com/docs/en (구 `docs.anthropic.com/en/docs/claude-code` 와 `docs.claude.com/en/docs/claude-code` 모두 301 리다이렉트)
> Changelog: https://code.claude.com/docs/en/changelog
