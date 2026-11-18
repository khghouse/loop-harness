# OpenAI가 공개한 에이전트 루프 및 하네스 설계 원칙 (2026-11-18 기준)

> 출처 확인 방식: openai.com/index/* 페이지(harness-engineering, unrolling-the-codex-agent-loop, agentic-ai-foundation)는 WebFetch에서 HTTP 403이 떠서 원문을 직접 열지 못함. 이 두 게시물에 대한 내용은 검색 스니펫과 2차 요약에서 가져왔고, 해당 항목마다 그렇게 표시함. cdn.openai.com의 "A practical guide to building agents" PDF는 받았지만 텍스트 추출 도구가 없어 읽지 못했고, 2차 요약을 사용함. Codex 문서와 Agents SDK 문서는 1차 출처에서 직접 읽음.
> 중요한 관찰: `developers.openai.com/codex/*` URL은 이제 `learn.chatgpt.com/docs/*`로 308(영구) 리다이렉트됨. 즉 Codex 문서가 옮겨졌고, 옛 링크는 리다이렉트로만 동작함. 문서 일부는 "ChatGPT Work"라는 이름도 언급함.

## Q1. "A practical guide to building agents"의 내용 (오케스트레이션, 가드레일, 도구, 사람 개입)

### Takeaway
OpenAI의 기본 권장안은 **단일 에이전트 + 도구 + 명시적 종료 조건이 있는 run loop**로 시작하고, 로직이 복잡해지거나 도구가 너무 많아질 때만 manager(agents-as-tools) 패턴이나 decentralized(handoff) 패턴으로 나누는 것. 가드레일은 여러 겹으로 쌓고, 실패 임계치를 넘거나 고위험 행동일 때 사람이 개입함.

### Cited Findings
- 발행: 2025년 4월 무렵 OpenAI CDN에 PDF로 공개됨. ibl.ai 요약 글의 날짜는 2025-06-16(원문 날짜가 아니라 요약 글 날짜). 원 PDF: https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf — [ibl.ai 요약](https://ibl.ai/blog/openai-a-practical-guide-to-building-agents.md); [Maginative](https://www.maginative.com/article/how-to-build-ai-agents-a-detailed-practical-guide-from-openai/)
- 에이전트를 만들 만한 경우는 세 가지: 판단 로직이 복잡하거나 규칙이 자주 바뀔 때, 비정형 데이터를 해석해야 할 때, 규칙 기반 자동화나 RPA가 엣지 케이스에서 깨질 때 — [ibl.ai](https://ibl.ai/blog/openai-a-practical-guide-to-building-agents.md)
- 구성 요소는 Model, Tools, Instructions 세 가지 — [ibl.ai](https://ibl.ai/blog/openai-a-practical-guide-to-building-agents.md)
- 모델 선택: 가장 성능 좋은 모델로 시작해 eval로 기준선을 잡은 뒤, 감당되는 부분부터 작은 모델로 바꿈 — [Aktagon 요약](https://cheat.aktagon.com/sheets/agents/)
- 도구는 세 종류: Data(정보 조회), Action(실행), Orchestration(다른 에이전트를 도구로 사용) — [Aktagon](https://cheat.aktagon.com/sheets/agents/)
- Instructions 작성법: 기존 SOP나 정책 문서를 바탕으로 하고, 작업을 단계로 쪼개 단계마다 구체적인 행동을 정하고, 엣지 케이스를 적어 둠. 변수가 들어간 프롬프트 템플릿을 씀 — [Aktagon](https://cheat.aktagon.com/sheets/agents/)
- 단일 에이전트 루프는 종료 조건이 충족될 때까지 돔. 종료 조건 예시는 final-output 도구 호출, 도구 호출 없는 응답, 에러, max turns — [Aktagon](https://cheat.aktagon.com/sheets/agents/)
- 멀티 에이전트는 단일 에이전트가 병목이 되거나 요구가 커질 때만 도입(start simple) — [ibl.ai](https://ibl.ai/blog/openai-a-practical-guide-to-building-agents.md)
- Manager 패턴: 중앙 에이전트가 전문 에이전트들을 도구로 호출하고 제어권과 컨텍스트를 유지함. Decentralized 패턴: 동료 에이전트끼리 단방향 handoff로 대화 상태를 넘기고 중앙 제어자가 없음 — [Aktagon](https://cheat.aktagon.com/sheets/agents/); [agentic-ai readthedocs](https://agentic-ai.readthedocs.io/en/latest/DesignPatterns/openai-patterns/)
- 가드레일 종류: relevance classifier, safety classifier(jailbreak/prompt injection), PII filter, moderation, 도구 위험도(low/medium/high), 규칙 기반(blocklist, 입력 길이 제한, regex), output validation. 여러 겹으로 쌓고, 데이터 프라이버시와 콘텐츠 안전부터 시작해 실제로 발생한 실패를 보며 추가함 — [Aktagon](https://cheat.aktagon.com/sheets/agents/)
- 사람 개입 트리거는 두 가지: 실패 임계치 초과(재시도 한도), 고위험 행동(환불, 결제 등) — [Aktagon](https://cheat.aktagon.com/sheets/agents/); [ibl.ai](https://ibl.ai/blog/openai-a-practical-guide-to-building-agents.md)

### Inferences
- loop-harness에 바로 옮길 수 있는 것: 루프 실행 단위마다 명시적인 종료 조건(완료 신호, max turns/iterations, 에러, 실패 임계치)을 두고, 고위험 행동은 승인 게이트로 막음. 이 가이드와 Agents SDK의 `max_turns`, 가드레일 tripwire와 같은 모델임.

### Gaps
- PDF 원문을 직접 읽지 못함(403은 아니었으나 추출 도구 없음). 정확한 문구, 쪽 번호, "optimistic execution" 같은 세부 표현, 도구 수에 대한 수치 임계는 확인하지 못함.

## Q2. OpenAI의 "harness engineering" 관련 공개 자료와 핵심 원칙

### Takeaway
"Harness engineering: leveraging Codex in an agent-first world"(2026-02-11, Ryan Lopopolo)와 "Unrolling the Codex agent loop"(2026-01-23, Michael Bolin)이 핵심 1차 자료. 메시지는 "사람은 방향을 잡고 에이전트가 실행한다"는 것. 엔지니어링의 중심은 환경, 피드백 루프, 제어 시스템을 설계하는 일로 옮겨감: 저장소를 단일 기록 원천으로 삼고, AGENTS.md는 약 100줄짜리 지도로 두고, 아키텍처는 linter와 구조 테스트로 기계적으로 강제하고, 앱과 관측 데이터를 에이전트가 읽을 수 있게 하고, 백그라운드 에이전트로 정기적인 "가비지 컬렉션"을 함.

### Cited Findings
**Harness engineering 게시물** (원문은 403으로 접근 불가. 내용은 2차 요약)
- 2026-02-11 공개, 저자 Ryan Lopopolo(OpenAI Codex 팀). 원문 URL: https://openai.com/index/harness-engineering/ — [검색 결과 / businessdatasolutions 요약](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- 수치: 약 5개월 동안 엔지니어 7명이 약 100만 줄, 약 1,500개 PR을 냄. 엔지니어 1인당 하루 PR 3.5개. 손으로 쓴 코드는 0줄(앱, 테스트, CI, 문서, 관측, 툴링 모두 Codex가 작성). Codex 한 번 실행이 최대 약 6시간 동안 단일 작업을 수행 — [businessdatasolutions 요약](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- 첫 커밋은 2025년 8월 말 빈 저장소에 들어감. 스캐폴드는 Codex CLI와 GPT-5로 생성 — [검색 요약](https://openai.com/index/harness-engineering/)
- 하네스의 정의: 에이전트가 할 수 있는 일을 제한하고, 무엇을 할지 알려 주고, 결과를 검증하고, 잘못되면 바로잡는 시스템. Context engineering은 "에이전트가 무엇을 봐야 하는가"를, harness engineering은 "시스템이 무엇을 막고, 측정하고, 교정해야 하는가"를 다룸(검색 요약에 나온 표현이라 원문 문구와 대조 필요) — [검색 요약](https://openai.com/index/harness-engineering/)
- 초기에 진척이 느렸던 이유는 Codex의 능력이 아니라 환경이 덜 정의돼 있었기 때문. 고정된 제약은 컴퓨트가 아니라 사람의 주의력 — [검색 요약](https://openai.com/index/harness-engineering/)
- **저장소가 단일 기록 원천**: 에이전트가 컨텍스트에서 볼 수 없는 것은 없는 것과 같음. 슬랙, 사람 머릿속, 외부 문서에 있는 지식은 버전 관리되는 markdown으로 저장소에 넣어야 함 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- **AGENTS.md는 지도이지 백과사전이 아님(약 100줄)**: 거대한 단일 AGENTS.md는 네 가지 이유로 실패함. 컨텍스트가 부족해지고, 지시가 너무 많으면 사실상 지시가 없는 것과 같아지고, 금방 낡고, 기계적으로 검증하기 어려움. 그래서 짧은 목차에서 `docs/` 트리로 연결하는 progressive disclosure 구조로 바꿈 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- `docs/` 구조: `design-docs/`(검증 상태, core beliefs), `exec-plans/`(active, completed, tech-debt tracker), `generated/`(DB 스키마 문서 등), `product-specs/`(인덱스 포함), `references/`(llms.txt 포함). 최상위 파일은 DESIGN, FRONTEND, PLANS, PRODUCT_SENSE, QUALITY_SCORE. 실행 계획을 일급 산출물로 버전 관리하고, 품질 등급으로 격차를 추적함 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- **에이전트가 읽을 수 있는 환경(legibility)**: worktree마다 앱 인스턴스를 따로 띄우고, Chrome DevTools Protocol로 DOM 스냅샷, 스크린샷, UI 조작을 함. worktree마다 임시 관측 스택(logs, metrics, traces)을 두고 LogQL, PromQL, TraceQL로 조회함. 검증 루프는 스냅샷 → UI 경로 실행 → 스냅샷 → 수정 → 재검증. 예시 성능 프롬프트는 "시작 800ms 이내", "핵심 사용자 여정 4개에서 2초 넘는 span 없음" — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- **기계적 강제**: 도메인마다 계층을 고정함(Types → Config → Repo → Service → Runtime → UI). 의존 방향은 엄격히 검증하고, 횡단 관심사는 단일 Providers 인터페이스로만 통과시킴. Codex가 만든 custom linter가 경계를 지키게 하고, 경계 안에서는 자유를 줌. 엄격한 구조는 비용이 아니라 드리프트 없이 빠르게 가기 위한 전제 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- linter 에러 메시지에 수정 방법을 넣는 것이 효과적이었음. 실패 메시지가 그대로 에이전트 컨텍스트로 들어가기 때문 — [검색 요약(여러 2차 출처)](https://gist.github.com/intellectronica/1a9018ed642096fc81b0eeb1f2c8b63c)
- **취향(taste) 피드백**: 리뷰 코멘트, 리팩터링 PR, 사용자 버그를 문서 업데이트나 lint 규칙으로 바꿈. 문서로 부족하면 규칙을 코드로 승격함 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- **머지 철학**: 머지 게이트는 최소화하고 차단은 드물게. PR 수명은 짧게 하고 flaky 테스트는 후속 실행으로 처리함. 에이전트 처리량이 사람의 주의력을 크게 넘으면 수정은 싸고 기다림은 비싸기 때문 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- **엔트로피와 가비지 컬렉션**: Codex는 저장소에 있는 패턴을 나쁜 것까지 복제하므로 드리프트가 쌓임. 초기에는 매주 금요일(업무 시간의 20%)을 "AI slop" 정리에 썼고, 이후 이를 자동화함. golden principles(공용 유틸 우선, 경계에서 데이터 검증 또는 typed SDK 사용 등)를 기계적 규칙으로 만들고, 백그라운드 Codex 작업이 편차를 스캔해 품질 등급을 갱신하고 작은 리팩터링 PR을 엶(대부분 1분 안에 리뷰하고 자동 머지). 기술 부채는 고금리 대출처럼 조금씩 계속 갚음. doc-gardening 에이전트가 낡은 문서를 찾아 수정 PR을 엶 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)
- **에이전트 간 리뷰와 end-to-end 자율성**: 프롬프트 하나로 버그 재현 → 실패 영상 녹화 → 수정 → 앱을 조작해 검증 → 영상 재녹화 → PR 생성 → 피드백 대응 → 빌드 실패 수정 → 판단이 필요할 때만 사람에게 escalation → 머지까지 진행. 리뷰는 대부분 에이전트 간에 이뤄지고 사람 리뷰는 선택. 저자는 이 결과가 이 저장소의 구조와 툴링에 크게 의존하며 일반화하기 어렵다고 단서를 붙임 — [businessdatasolutions](https://businessdatasolutions.github.io/ai-wiki/sources/2026-02-11-lopopolo-codex-harness-engineering)

**Unrolling the Codex agent loop** (원문 403. 검색 스니펫과 2차 자료 기반)
- 2026-01-23 공개, 저자 Michael Bolin. "Codex가 어떻게 동작하는가" 시리즈의 첫 글. URL: https://openai.com/index/unrolling-the-codex-agent-loop/ — [검색 요약](https://openai.com/index/unrolling-the-codex-agent-loop)
- 루프 구조: 추론 결과는 최종 응답이거나 도구 호출 요청임. 도구 호출이면 실행한 뒤 출력을 프롬프트에 붙여 다시 질의함. 사용자 메시지 하나가 한 turn이고, 한 turn 안에서 도구 호출이 수백 번 일어날 수 있어 컨텍스트 윈도우 관리가 하네스의 책임이 됨 — [검색 요약](https://openai.com/index/unrolling-the-codex-agent-loop)
- 프롬프트 구성: instructions(설정 파일 또는 모델 기본값), tools(Responses API 스키마), 그리고 입력 항목(AGENTS.md, 환경 컨텍스트, 사용자 메시지). sandbox 안내 메시지는 내장 shell 도구에만 적용되고, MCP 도구는 Codex가 샌드박싱하지 않으므로 MCP 서버가 스스로 가드레일을 걸어야 함 — [검색 요약](https://openai.com/index/unrolling-the-codex-agent-loop); [InfoQ](https://www.infoq.com/news/2026/02/codex-agent-loop)
- 프롬프트 캐싱: 이전 프롬프트가 새 프롬프트의 정확한 prefix가 되도록 대화를 선형으로 append만 함. 작업 디렉터리가 바뀌면 기존 내용을 고치지 않고 새 environment context 메시지를 추가함. 대화 도중 AGENTS.md를 바꾸면 캐시가 깨짐. 무상태 설계로 `previous_response_id`를 쓰지 않고 매번 전체 기록을 보냄(ZDR 호환). auto_compact_limit을 넘으면 `/responses/compact` 엔드포인트로 자동 압축하고, 압축하면 prefix가 바뀌어 캐시가 리셋됨 — [InfoQ](https://www.infoq.com/news/2026/02/codex-agent-loop); [codex.danielvaughan.com(3rd party)](https://codex.danielvaughan.com/2026/04/21/codex-cli-prompt-caching-maximise-cache-hits-cost-reduction/). 압축 트리거 기준은 출처마다 다름(약 90% 대 설정 가능한 임계치).

### Inferences
- loop-harness 설계에 바로 쓸 수 있는 원칙:
  1. 저장소 안의 문서와 계획 파일(`docs/exec-plans/active` 등)을 루프의 상태 저장소로 삼음.
  2. AGENTS.md는 짧은 지도로 두고 상세 문서로 연결함.
  3. 검증은 사람의 리뷰가 아니라 linter, 구조 테스트, 관측 쿼리 같은 기계적 신호로 함.
  4. 에러 메시지에 수정 방법을 넣음.
  5. 정기적인 GC나 doc-gardening 루프를 일급 기능으로 둠.
  6. append-only 대화 기록과 안정적인 prefix로 캐시 효율을 지킴.
- "Ralph Wiggum loop"라는 이름은 OpenAI 원문의 용어가 아니라 2차 출처가 붙인 라벨로 보임. 원문은 "리뷰어가 만족할 때까지 에이전트가 반복한다"고 서술함.

### Gaps
- openai.com 원문 두 편을 직접 읽지 못해 정확한 인용, 수치(PR 수 등), 시리즈 후속편 존재 여부를 확인하지 못함. 2차 요약끼리 수치가 일부 다름(예: "1인당 하루 3.5 PR"은 시간이 갈수록 늘었다는 설명이 붙어 있음).

## Q3. Codex(CLI, cloud, IDE)의 구조: AGENTS.md, 샌드박스와 승인, skills, MCP, exec, code review, hooks

### Takeaway
Codex는 계층형 AGENTS.md(32KiB 상한), OS 수준 샌드박스와 승인 정책, Agent Skills 표준 기반의 skills(progressive disclosure), hooks(Claude Code와 비슷한 이벤트 모델이며 Claude 플러그인 변수도 호환), `codex exec`(JSONL 이벤트 스트림, 출력 JSON Schema 지정), `/review`와 GitHub 자동 리뷰로 구성됨. 문서는 2026년에 learn.chatgpt.com으로 옮겨짐.

### Cited Findings
- **AGENTS.md 탐색**: 실행(또는 TUI 세션)당 한 번 체인을 만듦. 전역(`~/.codex` 또는 `CODEX_HOME`)에서는 `AGENTS.override.md`가 있으면 그것을, 없으면 `AGENTS.md`를 씀. 프로젝트에서는 Git 루트부터 현재 디렉터리까지 내려가며 디렉터리마다 최대 한 파일을 고름(우선순위: `AGENTS.override.md` → `AGENTS.md` → `project_doc_fallback_filenames`). 루트에서 현재 디렉터리 순으로 이어 붙이므로 가까운 파일이 나중에 와서 우선함. 합계가 `project_doc_max_bytes`(기본 32 KiB)에 도달하면 더 추가하지 않음 — [Codex docs: AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- **샌드박스 모드**(`sandbox_mode`): `read-only`, `workspace-write`(로컬 작업의 저마찰 기본값), `danger-full-access`. 승인 정책(`approval_policy`): `on-request`, `never`. **`untrusted`는 폐지됨**(superseded). `approvals_reviewer = auto_review`로 승인 요청을 리뷰어 에이전트에게 보낼 수 있고, 이 설정은 샌드박스 경계를 바꾸지 않음. OS 메커니즘은 macOS Seatbelt, Linux bubblewrap, Windows 전용 샌드박스. 권장 로컬 기본값은 `workspace-write` + `on-request`. 쓰기 경로 확장은 `writable_roots`로, 예외는 명령 prefix 규칙(rules)으로 처리함 — [Codex docs: Sandboxing](https://learn.chatgpt.com/docs/sandboxing)
- **`codex exec`(비대화형/CI)**: 진행 상황은 stderr로, 최종 메시지만 stdout으로 감. `--json`이면 JSONL 이벤트(`thread.started`, `turn.started/completed/failed`, `item.*`, `error`)를 내보내고, item 종류는 agent message, reasoning, command execution, file change, MCP tool call, web search, plan update. `--output-schema`로 최종 출력 JSON Schema를 강제할 수 있고 `-o`는 최종 메시지를 파일로 저장함. exec의 기본 샌드박스는 read-only. **`--full-auto`는 더 이상 지원하지 않음**(대체: `--sandbox workspace-write` 또는 `--approve-for-me`). 그 밖에 `--ephemeral`, `codex exec resume --last`, `--trust`, `--ignore-rules`가 있음. 프로젝트 설정과 MCP는 신뢰한 프로젝트에서만 로드됨. GitHub Actions에서는 `openai/codex-action` 사용을 권장하고, 패치를 만드는 job(read 권한)과 PR을 여는 job(write 권한, API 키 없음)을 분리하는 예시를 듦 — [Codex docs: Non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode)
- **Hooks**: 이벤트는 `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PreCompact`, `PostCompact`, `SubagentStart/Stop`, `Stop`, `SessionEnd`, `PreModelRequest/PostModelRequest`(진단 전용), `Interrupt`. 설정 위치는 `/etc/codex/hooks.json`, `~/.codex/hooks.json`, `.codex/hooks.json`(신뢰한 프로젝트만), `config.toml`의 `[hooks]`, 플러그인, `requirements.toml`(관리형). 훅은 stdin으로 JSON을 받음. exit 2는 차단, JSON 출력으로 `permissionDecision: "deny"`나 `updatedInput`을 줄 수 있음. `Stop` 이벤트에서 `decision:"block"`을 주면 reason이 새 프롬프트가 되어 작업이 계속됨(루프 지속에 쓰임). 훅은 해시 기반으로 신뢰를 관리하고 기본 타임아웃은 600초. 플러그인 훅에는 호환성을 위해 `CLAUDE_PLUGIN_ROOT`/`CLAUDE_PLUGIN_DATA`도 주입됨 — [Codex docs: Hooks](https://learn.chatgpt.com/docs/hooks)
- **Skills**: `SKILL.md`(frontmatter에 `name`, `description` 필수)와 선택 디렉터리 `scripts/`, `references/`, `assets/`, `agents/openai.yaml`로 구성됨. Progressive disclosure로 처음에는 이름, 설명, 경로만 로드하고(기본 컨텍스트의 2% 또는 8,000자) 선택된 뒤에 본문을 읽음. 위치는 `.agents/skills`(현재 디렉터리에서 저장소 루트까지), `$HOME/.agents/skills`, `/etc/codex/skills`, 시스템 내장. "open agent skills standard"(agentskills.io)를 기반으로 함. 호출은 명시적(`$skill-name`, `/skills`) 또는 description 매칭에 따른 암묵적 호출 — [Codex docs: Build skills](https://learn.chatgpt.com/docs/build-skills)
- **Code review**: CLI의 `/review`는 별도 리뷰어를 띄워 작업 트리를 바꾸지 않고 우선순위가 매겨진 발견 사항을 보고함(base 브랜치, 미커밋 변경, 특정 커밋, 커스텀 지시 중 선택). GitHub/GitLab에서는 `@codex review`로 호출하고, 자동 리뷰를 켤 수 있으며, 리뷰 지침은 설정이나 skills로 줌 — [Codex docs: Code review](https://learn.chatgpt.com/docs/code-review)
- 문서 내비게이션에 MCP(`/codex/extend/mcp`), Cloud(`/codex/cloud`, cloud environments) 항목이 있음 — [Codex docs nav](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- MCP 도구는 Codex 샌드박스 밖에서 실행됨(위 Q2 참조) — [검색 요약](https://openai.com/index/unrolling-the-codex-agent-loop)

### Inferences
- loop-harness가 도구에 종속되지 않으려면 공통분모를 맞추는 것이 현실적임: AGENTS.md(Codex는 기본으로 읽음), `.agents/skills/*/SKILL.md`(Agent Skills 표준), hooks의 이벤트 모델(Codex가 Claude Code 이벤트 이름과 플러그인 변수 호환을 의식적으로 맞추고 있음), 비대화형 실행의 JSONL 이벤트 스트림, 출력 스키마. Go 백엔드는 `codex exec --json` 이벤트를 그대로 수집하는 어댑터를 쓸 수 있음.
- `Stop` 훅의 block 기능이 Codex 공식 기능으로 존재함. 즉 "완료 조건을 만족할 때까지 루프를 계속 돌리는" 방식(Ralph 스타일)이 하네스 수준에서 표준 기능으로 지원됨.

### Gaps
- MCP 설정(`config.toml`의 `[mcp_servers]`), Codex Cloud 환경, 병렬 작업, best-of-N의 세부 내용은 이번에 직접 확인하지 못함. "ChatGPT Work"와 Codex의 관계(브랜드 통합 여부)도 확인하지 못함.

## Q4. OpenAI Agents SDK / Responses API: 루프, handoffs, guardrails, tracing, evals

### Takeaway
Agents SDK의 Runner 루프는 "LLM 호출 → 최종 출력이면 종료 / handoff면 에이전트 교체 / 도구 호출이면 실행 후 반복"을 `max_turns`로 제한하는 구조. 가드레일은 입력, 출력, 도구 세 단계에 있고 tripwire로 실행을 중단함. HITL(사람 개입)은 `RunState`를 직렬화해 멈췄다가 다시 시작하는 방식. 평가는 trace grading과 trace evals로 결과뿐 아니라 실행 과정 자체를 채점함.

### Cited Findings
- Runner 루프: LLM 호출 → (a) 최종 출력이면 종료 (b) handoff면 현재 에이전트와 입력을 교체하고 반복 (c) 도구 호출이면 실행하고 결과를 붙여 반복. 최종 출력은 도구 호출이 없고 `output_type`에 맞는 출력. `max_turns`를 넘으면 `MaxTurnsExceeded`가 발생하고, 에러 핸들러로 통제된 최종 출력을 낼 수 있음 — [Agents SDK: Running agents](https://openai.github.io/openai-agents-python/running_agents/)
- 상태 관리는 네 가지: `to_input_list()`(클라이언트 관리), Sessions(SDK가 자동 저장), `conversation_id`(Conversations API), `previous_response_id`(서버 체이닝) — [Agents SDK](https://openai.github.io/openai-agents-python/running_agents/)
- 장기 실행과 내구성: Temporal, Dapr, Restate, DBOS 연동. HITL은 도구 승인 대기에서 멈추고 저장된 `RunState`로 다시 시작함. `cancel(mode="after_turn")`도 있음 — [Agents SDK](https://openai.github.io/openai-agents-python/running_agents/)
- 가드레일: 입력 가드레일은 첫 에이전트에서만 돌고 병렬(기본) 또는 블로킹으로 실행됨. 블로킹이면 토큰 소비나 도구 실행 전에 막을 수 있음. 출력 가드레일은 마지막 에이전트의 최종 출력에 적용됨. 도구 가드레일은 매 호출 전후로 실행됨(MCP 도구 포함, hosted tool과 handoff는 제외). tripwire가 걸리면 `*TripwireTriggered` 예외로 중단함 — [Agents SDK: Guardrails](https://openai.github.io/openai-agents-python/guardrails/)
- Trace grading: 에이전트 trace(결정, 도구 호출, 추론)에 구조화된 점수나 라벨을 붙임. 여러 사례에 걸친 trace eval로 회귀를 잡음. 블랙박스 평가보다 "왜" 실패했는지를 알려 줌. 대시보드 흐름은 Logs > Traces → grader 생성 → "Grade all" → eval 대시보드. Agent Builder 워크플로의 trace는 "전환 기간"에만 지원된다고 명시함(Agent Builder가 단계적으로 정리되고 있다는 신호일 수 있음) — [OpenAI docs: Trace grading](https://developers.openai.com/api/docs/guides/trace-grading)

### Inferences
- loop-harness의 실행 모델은 SDK 루프와 같은 구조(turn 단위, 최대 반복, 종료 판정, 가드레일 tripwire, 일시정지/재개 가능한 상태 직렬화)를 따르는 것이 업계 표준에 가까움. 관측은 trace 단위로 남기고, 사후에 trace grader로 채점할 수 있게 설계하는 것이 OpenAI 방향과 맞음.

### Gaps
- Agents SDK의 tracing 기본값(자동 활성화, 내보내기 프로세서)과 handoff 문서는 이번에 직접 읽지 않음. AgentKit/Agent Builder의 현재 상태(전환 또는 폐지 여부)는 trace grading 문서의 "transition window" 문구만으로는 확정할 수 없음.

## Q5. 오픈 표준에서 OpenAI의 역할 (AGENTS.md, AAIF, MCP, Agent Skills)

### Takeaway
OpenAI는 AGENTS.md를 만들어 2025-12-09에 Linux Foundation 산하 Agentic AI Foundation(AAIF)에 기증함(공동 설립: OpenAI, Anthropic, Block. MCP와 goose도 함께 기증됨). Codex는 MCP를 지원하고, skills는 agentskills.io 오픈 표준 위에 만들어져 있음. 사실상 "AGENTS.md + MCP + Agent Skills"가 업계 공통 기반임.

### Cited Findings
- AAIF는 2025-12-09 Linux Foundation 산하에서 출범. OpenAI가 AGENTS.md를, Anthropic이 MCP를, Block이 goose를 기증함. Google, Microsoft, AWS, Bloomberg, Cloudflare 등이 지원함. 목적은 중립적 거버넌스와 상호운용성 — [OpenAI: Agentic AI Foundation](https://openai.com/index/agentic-ai-foundation/); [RCPmag](https://rcpmag.com/Articles/2025/12/09/Linux-Foundation-Unites-Major-Tech-Firms.aspx); [Fierce Network](https://www.fierce-network.com/newswire/linux-foundation-forms-agentic-ai-foundation)
- Linux Foundation 발표 기준으로 공개된 MCP 서버는 1만 개 이상(단일 출처 수치) — [검색 요약 / IT Brief](https://itbrief.co.uk/story/linux-foundation-unveils-agentic-ai-standards-group)
- Codex skills는 "open agent skills standard"(agentskills.io)를 기반으로 하고 `.agents/skills` 경로를 씀 — [Codex docs: Build skills](https://learn.chatgpt.com/docs/build-skills)
- Codex 훅은 Claude 플러그인 훅 호환 변수(`CLAUDE_PLUGIN_ROOT`)를 제공함 — [Codex docs: Hooks](https://learn.chatgpt.com/docs/hooks)

### Inferences
- loop-harness가 "업계 주류"를 따르려면 AGENTS.md(AAIF 표준), MCP(AAIF 표준), Agent Skills(agentskills.io)를 기본 형식으로 쓰고, 에이전트별 설정(CLAUDE.md, `.codex/`)은 어댑터 계층으로 처리하는 것이 맞음.

### Gaps
- AGENTS.md 명세 자체(agents.md 사이트)와 AAIF 이후 거버넌스 변화(새 프로젝트 편입, Agent Skills가 AAIF에 편입됐는지)는 이번 조사에서 확인하지 못함. openai.com AAIF 게시물 원문도 403으로 읽지 못함.
