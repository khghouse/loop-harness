# 업계 실무: 에이전트 루프 및 코딩 에이전트 하네스 설계 (Anthropic/OpenAI/Google 제외), 기준일 2026-11-18

출처 표기 규칙: [공식] = 회사 공식 엔지니어링 블로그/문서, [개인] = 개인 실무자 의견/블로그, [2차] = 애그리게이터/제3자 요약(검증 약함).

## Q1. 실무자/사상가 프레임워크 (Fowler/Thoughtworks, Böckeler, Willison, Huntley, Hashimoto, Beck, HumanLayer/Horthy, Osmani)

### Takeaway
2025~2026년 실무 담론의 중심 용어는 "harness engineering"이다. 공식은 **Agent = Model + Harness**이고, 하네스는 피드포워드 **guides**(AGENTS.md, specs, skills)와 피드백 **sensors**(lint, type, test, review agent)로 구성된다. 에이전트가 실수를 반복하면 사람은 하네스 자체를 반복 개선하는 "steering loop"를 돈다. 이 공통 골격을 Ralph loop(Huntley), Research→Plan→Implement(Horthy), 12-factor agents, spec-driven development(SDD)가 각기 다른 각도에서 구체화한다.

### Cited Findings
**Birgitta Böckeler / martinfowler.com: "Harness Engineering for Coding Agent Users" (2026-04-02, 2026-02-17 짧은 메모 후속) [개인, Thoughtworks Distinguished Engineer]**
- "하네스"는 모델을 제외한 에이전트의 모든 것이다. 이 글은 그중 **user harness**에 초점을 둔다. 도구가 기본 제공하는 하네스(system prompt, retrieval, orchestration) 위에 개발자가 직접 쌓는 바깥쪽 통제를 말한다. 목표는 두 가지다. 첫 시도 성공 확률을 높이고, 사람 리뷰 전에 에이전트가 스스로 교정하게 하는 것이다. 그만큼 리뷰 수고와 토큰 낭비가 줄어든다 — [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)
- **Guides (feedforward)**: 행동 전에 조향한다. 예: AGENTS.md의 코딩 컨벤션, skills, codemods. **Sensors (feedback)**: 행동 후에 관찰해 자기교정을 가능하게 한다. 특히 "고치는 방법"을 담은 LLM용 커스텀 linter 메시지가 효과적이다. 피드백만 있으면 같은 실수를 반복하고, 피드포워드만 있으면 규칙이 검증되지 않는다 — [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)
- **Computational vs Inferential**: computational은 결정적이고 빠른(ms~s) 검사다(tests, linters, type checkers, 구조 분석). 매 변경마다 돌린다. inferential은 AI 코드리뷰나 LLM-as-judge처럼 의미적 판단을 하는 검사로, 느리고 비싸며 비결정적이다 — [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)
- 규제 범주는 셋이다. (1) **Maintainability harness**: 지금 가장 쉽다. (2) **Architecture fitness harness**: fitness function 기반이다. (3) **Behaviour harness**: 가장 어렵고("the elephant in the room"), AI가 생성한 테스트를 너무 믿는 문제가 있다 — [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)
- **"Keep quality left"**: 비용과 속도에 따라 검사를 시점별로 나눈다. 커밋 전에는 빠른 lint, 빠른 test, 기본 리뷰 에이전트를 돌린다. 통합 후 파이프라인에서는 mutation testing과 아키텍처 리뷰를 돌린다. 상시로는 drift/health sensor(dead code, 의존성 스캔, SLO)를 둔다 — [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)
- 그 밖의 개념: **steering loop**(이슈가 재발하면 guide나 sensor를 개선), **harnessability**(강타입과 명확한 모듈 경계가 하네스를 쉽게 만든다), **ambient affordances**(Ned Letcher), **harness templates**(서비스 토폴로지별 guide+sensor 번들), Ashby의 필수다양성 법칙. 레거시 코드베이스는 하네스가 가장 필요하지만 만들기는 가장 어렵다 — [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)
- 미해결 질문: guide와 sensor 간 모순을 어떻게 관리할지, 하네스 커버리지와 품질을 어떻게 측정할지(code coverage에 해당하는 지표), sensor가 조용한 것이 품질이 좋아서인지 탐지가 약해서인지 — [martinfowler.com](https://martinfowler.com/articles/harness-engineering.html)

**Böckeler: "Understanding Spec-Driven-Development: Kiro, spec-kit, and Tessl" (2025-10) [개인]**
- SDD를 세 수준으로 나눈다. **spec-first**(코딩 전에 spec 작성), **spec-anchored**(작업 후에도 spec을 살아있는 산출물로 유지), **spec-as-source**(spec이 원천이고 코드는 파생물). 조사한 도구는 모두 spec-first였다. spec 포맷 표준은 없고, 생성이 비결정적이어서 spec과 코드 사이 drift가 생긴다. Kiro는 단순 버그 수정에도 user story 4개와 수용기준 16개를 만들었다는 과잉 사례가 보고됐다 — [martinfowler.com](https://www.martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html) (세부 내용은 [2차](https://engineered.at/articles/understanding-spec-driven-development-kiro-spec-kit-and-tessl) 요약 기반)

**Thoughtworks Technology Radar Vol. 34 (2026-04) [공식, 단 Thoughtworks 스스로 "한 회사의 관점"이라고 밝힘]**
- **Curated shared instructions for software teams**: CLAUDE.md, AGENTS.md, .cursorrules를 서비스 스캐폴딩용 baseline 저장소에 넣어 팀 전체에 배포한다 — [Thoughtworks Radar](https://www.thoughtworks.com/radar/techniques/curated-shared-instructions-for-software-teams)
- **GitHub Spec Kit** 항목: "constitution"으로 범위, 코딩 표준, 아키텍처를 고정한다. 한 팀의 spec→plan→tasks→coding→review 수명주기가 보고돼 있다 — [Thoughtworks Radar](https://www.thoughtworks.com/en-au/radar/languages-and-frameworks/github-spec-kit)
- Vol.34의 중심 테마는 **cognitive debt**(시스템이 하는 일과 팀의 이해 사이의 격차)다. Agent Skills, SDD 프레임워크, curated instructions는 "feedforward" 통제로 분류된다. AGENTS.md는 쌓이면서 서로 충돌하고 묻힌 내용은 무시될 수 있으니 선별적으로 유지하라고 권한다. "SDD"와 "harness engineering" 용어가 혼용·중첩되고 있다고도 지적한다 — [2차 요약](https://agentic-ai.readthedocs.io/en/latest/AgenticTechStack/thoughtworks-radar-vol34/), [itbrief](https://itbrief.news/story/ai-coding-boom-deepens-cognitive-debt-says-thoughtworks)
- Thoughtworks 팟캐스트 "What is harness engineering?" — [thoughtworks.com](https://www.thoughtworks.com/insights/podcasts/technology-podcasts/what-harness-engineering)

**Simon Willison: "Designing agentic loops" (2025-09-30) [개인]**
- 에이전트 정의는 "tools in a loop to achieve a goal"이다. agentic loop 설계를 새로운 핵심 스킬로 본다. 루프에 적합한 문제는 **명확한 성공 기준이 있고 시행착오가 필요한 문제**다(실패 테스트 디버깅, 성능 튜닝, 의존성 업그레이드, 컨테이너 이미지 축소). 좋은 테스트 스위트가 에이전트의 가치를 크게 높인다 — [simonwillison.net](https://simonwillison.net/2025/Sep/30/designing-agentic-loops)
- **YOLO mode**(전부 자동 승인)의 위험은 파괴적 명령, secret이나 코드 유출(exfiltration), 공격 프록시화이며, prompt injection이 이를 유발할 수 있다. 대책은 컨테이너나 원격 sandbox(Codespaces 등), 신뢰 호스트 allowlist로 egress 제한, 스테이징 전용 credential, 예산 상한이 걸린 키(예: $5 한도의 전용 클라우드 조직)다. MCP보다 shell/CLI 도구를 선호하고, AGENTS.md에 도구별 예시 한 줄씩을 적는다 — [simonwillison.net](https://simonwillison.net/2025/Sep/30/designing-agentic-loops)

**Geoffrey Huntley: "Ralph Wiggum as a software engineer" (2025-07-14) [개인]**
- 가장 순수한 형태는 `while :; do cat PROMPT.md | claude-code ; done`다. 도구에 구애받지 않으며(tool-agnostic), 매 루프는 새 컨텍스트로 시작한다. **루프당 한 항목**만 처리한다. `specs/`(파일당 spec 하나)와 `fix_plan.md`(우선순위 TODO)를 매 루프 결정적으로 로드한다. plan은 자주 버리고 planning 실행으로 다시 만든다. `AGENT.md`(빌드와 실행 방법)는 "루프의 심장"이며 Ralph가 직접 갱신한다 — [ghuntley.com/ralph](https://ghuntley.com/ralph/)
- **Backpressure**: 테스트(방금 바꾼 단위만), 빌드, 타입체커와 정적분석으로 나쁜 출력을 빨리 거부한다. 메인 컨텍스트는 스케줄러 역할을 한다. 검색과 쓰기는 여러 subagent에 병렬로 맡기되 빌드와 테스트 검증은 subagent 하나만 돌린다. 쓸 수 있는 컨텍스트는 약 170k이며 최소한으로 사용한다 — [ghuntley.com/ralph](https://ghuntley.com/ralph/)
- **Signs 튜닝**: 실수가 반복되면 놀이터 표지판처럼 교정 지시를 추가한다. 실패 모드로는 검색 오판으로 인한 중복 구현, placeholder 구현("no placeholders" 지시로 대응), 잘못된 방향, 깨진 코드베이스(`git reset --hard` 또는 구조 프롬프트), 끝나지 않는 루프가 있다 — [ghuntley.com/ralph](https://ghuntley.com/ralph/)
- 후속 글 "everything is a ralph loop"(2026-01). Huntley는 CURSED 언어를 약 3개월 동안 루프로 만들었다 — [2차: latent.space wiki](https://latent.space/wiki/geoffrey-huntley/)

**Mitchell Hashimoto: "My AI Adoption Journey" (2026-02-05) [개인]**
- 단계는 다음과 같다. Drop the Chatbot → Reproduce Your Own Work → End-of-Day Agents → Outsource the Slam Dunks → **Engineer the Harness** → Always Have an Agent Running — [mitchellh.com](https://mitchellh.com/writing/my-ai-adoption-journey)
- Engineer the Harness의 내용: 에이전트가 실수할 때마다 AGENTS.md에 한 줄씩 추가한다("each line ... based on a bad agent behavior"). 스크린샷, 필터링된 테스트 실행 같은 전용 스크립트와 도구를 만들고 AGENTS.md에 기록한다. 에이전트가 틀렸을 때 바로 알려주는 빠르고 질 좋은 도구를 제공한다. 검증 수단을 주면 에이전트가 스스로 고친다 — [mitchellh.com](https://mitchellh.com/writing/my-ai-adoption-journey)
- 백그라운드 triage 에이전트는 **read-only**로 리포트만 만들고 GitHub에는 응답하지 않는다. 알림은 끄고 사람이 정한 시점에만 확인한다 — [mitchellh.com](https://mitchellh.com/writing/my-ai-adoption-journey)

**Kent Beck: "Augmented Coding: Beyond the Vibes" (2025) [개인]**
- augmented coding은 vibe coding과 다르다. 코드, 복잡도, 테스트, 커버리지를 신경 쓰고 손코딩과 같은 가치체계를 따른다. B+ tree 프로젝트에서 "genie"에게 TDD를 강제했다. 처음 두 시도는 복잡도가 쌓여 stall했고, 설계를 적극적으로 유도하자 개선됐다 — [newsletter.kentbeck.com](https://newsletter.kentbeck.com/p/augmented-coding-beyond-the-vibes)

**HumanLayer / Dex Horthy [회사 블로그이자 개인 방법론]**
- **12-Factor Agents** (2025-04): 주요 요소는 Own your context window(F3), Tools are structured outputs(F4), Own your control flow(F8, 프레임워크에 루프를 맡기지 말고 tool 선택과 실행 사이에 사람 승인을 끼울 수 있게), Compact errors into context(F9), Small focused agents(F10, 3~20 스텝으로 출처마다 다름), Trigger from anywhere(F11), Stateless reducer(F12) — [humanlayer.dev](https://humanlayer.dev/blog/12-factor-agents), 세부는 [2차](https://understandingdata.com/posts/12-factor-agents/)
- **Advanced Context Engineering for Coding Agents** (2025-08): **Research → Plan → Implement (RPI)**. 각 단계를 새 컨텍스트에서 실행하고, 진행 상태는 plan 파일에 compaction한다. "LLM은 stateless 함수이고 출력 품질은 입력 품질뿐"이라는 관점이다 — [humanlayer.dev](https://humanlayer.dev/blog/advanced-context-engineering)
- 이후 RPI는 7단계 "CRISPY"(Questions, Research, Design, Structure/Outline, Plan, Work, PR)로 진화했다고 한다. 거대한 plan이 instruction budget을 넘어 리뷰가 불가능해지는 문제 때문이다 — [2차: ZenML](https://www.zenml.io/llmops-database/evolution-from-rpi-to-crispy-multi-stage-workflow-for-production-coding-agents) (미검증)

### Inferences
- 모든 프레임워크에 공통으로 나오는 것: (1) 저장소에 커밋된 instruction 파일(AGENTS.md), (2) 결정적 backpressure/sensor, (3) 실수→하네스 수정이라는 피드백 루프, (4) 컨텍스트 최소화와 단계별 새 컨텍스트, (5) spec/plan 파일을 루프 간 상태로 사용. loop-harness는 이것들을 1급 객체(guide, sensor, plan artifact, iteration)로 모델링할 수 있다.
- Böckeler의 guides/sensors × computational/inferential 2×2 매트릭스와 "keep quality left" 시점 분류는 tool-agnostic 하네스의 설정 스키마로 그대로 쓰기 좋다.
- 레거시 프로젝트에 하네스가 가장 필요하지만 가장 어렵다는 지적은 사용자의 회사 프로젝트 경험과 직접 맞닿는다.

### Gaps
- Addy Osmani의 agentic engineering 관련 글은 이번 검색에서 1차 출처를 확보하지 못했다.
- "loop engineering"이라는 용어 자체의 기원과 정의를 담은 권위 있는 1차 출처를 찾지 못했다(개인 노트 [dsebastien](https://notes.dsebastien.net/30+Areas/33+Permanent+notes/33.02+Content/Loop+Engineering) 정도만 확인). 업계에서는 harness engineering, context engineering, agentic loop가 더 표준적인 용어로 보인다.
- Thoughtworks Radar Vol.35(2026년 하반기)가 나왔는지는 확인하지 못했다.

## Q2. 회사 엔지니어링 포스트 (GitHub, Microsoft, Stripe, Cursor, Cognition, Factory, Amp 등)

### Takeaway
프로덕션 사례는 **격리된 사전 준비 sandbox**, **결정적 노드와 에이전트 노드를 섞은 코드 정의 워크플로**, **shift-left lint/test**, **CI 재시도 횟수 상한**, **사람의 최종 PR 리뷰**라는 공통 패턴으로 수렴한다. 대규모 장기 실행에서는 planner/worker/judge 계층 구조가 표준이 되어가고 있다.

### Cited Findings
**Stripe: Minions (Part 1 2026-02-09, Part 2 2026-02-19) [공식]**
- 무인으로 one-shot end-to-end 작업을 하는 에이전트다. 주당 1,300개 이상의 PR이 머지되며(Part 1 시점은 1,000개 이상), 사람이 쓴 코드는 0이고 리뷰는 전부 사람이 한다 — [stripe.dev Part 2](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2), [Part 1](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents)
- **Devbox**: AWS EC2 인스턴스이며 warm pool로 약 10초 안에 준비된다(repo clone, Bazel/타입체크 캐시를 미리 데워 둠). 프로덕션 데이터, 실사용자 데이터, 임의 egress에서 격리된 QA 환경이다. blast radius가 제한되므로 에이전트는 확인 프롬프트 없이 full permission으로 실행한다 — [stripe.dev](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- 하네스는 Block의 오픈소스 **goose**를 fork한 것이다(2024년 말). 인터럽트 같은 사람용 기능은 제거했다 — 같은 출처
- **Blueprints**: 코드로 정의한 워크플로다. **deterministic node**("Run configured linters", "Push changes")와 **agentic node**("Implement task", "Fix CI failures")를 섞는다. 팀별 커스텀 blueprint(예: 마이그레이션)도 가능하다 — 같은 출처
- **Rule files**: 디렉터리나 파일 패턴 단위로 범위를 정하고, 에이전트가 파일시스템을 탐색할 때 붙는다. global rule은 컨텍스트 보호를 위해 최소화한다. Cursor rule 포맷을 쓰고 Claude Code 포맷으로 동기화한다 — 같은 출처
- **Toolshed**: 사내 중앙 MCP 서버에 도구가 약 500개 있다. minion에게는 기본적으로 작게 큐레이션한 부분집합만 주고, 파괴적 동작은 보안 통제 프레임워크가 차단한다 — 같은 출처
- **피드백 루프**: 테스트 300만 개 이상, pre-push hook, 백그라운드 lint daemon, push 전 로컬 lint 루프가 있다. **CI는 최대 2라운드**다. 첫 CI 후 autofix를 적용하고, 남은 실패는 agent node가 한 번 고친 뒤 2차 CI를 돌리고 사람에게 넘긴다. 토큰과 CI 시간을 아끼려고 shift-left한다 — 같은 출처

**Cursor: "Scaling long-running autonomous coding" (Wilson Lin, 2026-01-14) [공식]**
- 수백 개의 동시 에이전트를 수 주간 돌려 100만 줄 이상을 만들고 수조 토큰을 썼다. 시도한 구조는 flat 자기조정(위험 회피 행동이 생김), locking(에이전트 20개가 2~3개처럼 동작), optimistic concurrency를 거쳐 **Planners(재귀적 sub-planner) / Workers(단일 task, 상호 조정 없음) / Judge(사이클 경계에서 계속할지 판정, 다음 사이클은 새로 시작)** 구조로 정착했다 — [cursor.com](https://cursor.com/blog/scaling-agents)
- 실패해서 제거한 것: integrator 역할(오히려 병목이었다). 구조가 너무 적으면 충돌과 중복이 생기고 너무 많으면 취약해진다. "prompts matter more"라고 해서 하네스와 모델보다 프롬프트의 영향이 크다고 봤다. 역할별로 모델을 다르게 배정한다. drift와 tunnel vision을 막으려면 주기적인 fresh start가 필요하다 — [cursor.com](https://cursor.com/blog/scaling-agents)
- 2026-02-12 long-running agents 리서치 프리뷰를 출시했다 — [cursor.com](https://cursor.com/blog/long-running-agents)

**GitHub: Copilot coding agent (현재 문서에서는 "cloud agent") [공식 문서/블로그]**
- 세션마다 GitHub Actions 기반의 일회성(ephemeral) 격리 환경을 만들고 종료 후 폐기한다. 기본 **firewall**로 인터넷 접근을 제한해 유출을 막는다. 작업은 [WIP] draft PR로 추적한다. 에이전트는 PR을 "Ready for review"로 바꾸거나 approve, merge할 수 없다. 에이전트가 만든 PR이 트리거한 Actions workflow는 write 권한자의 승인이 있어야 실행된다 — [docs.github.com](https://docs.github.com/copilot/concepts/coding-agent/about-copilot-coding-agent), [risks-and-mitigations](https://docs.github.com/en/enterprise-cloud@latest/copilot/concepts/agents/cloud-agent/risks-and-mitigations), [github.blog 101](https://github.blog/ai-and-ml/github-copilot/github-copilot-coding-agent-101-getting-started-with-agentic-workflows-on-github/)

**GitHub/Microsoft: Spec Kit (2025-09 공개, MIT) [공식]**
- `specify init`으로 시작하며, 명령은 `/specify` → `/plan` → `/tasks` → implement 순이다(신버전은 `speckit.` 접두사를 쓰고 clarify/analyze 게이트가 붙는다). `.specify/` 템플릿과 **constitution** 파일을 쓴다. Copilot, Claude Code, Gemini CLI 등 에이전트에 구애받지 않는다. GitHub 스스로 "experiment"라고 부른다 — [github.github.com/spec-kit](https://github.github.com/spec-kit/index.html), [Microsoft Dev Blog 2025-09-15](https://developer.microsoft.com/blog/spec-driven-development-spec-kit/), [Visual Studio Magazine](https://visualstudiomagazine.com/articles/2025/09/03/github-open-sources-kit-for-spec-driven-ai-development.aspx)

**Cognition (Devin) [공식 문서, 2차 경유]**
- **Dynamic Workflows**(2026-10-13 발표): 결정적인 Python 스크립트가 Devin 에이전트 팀을 오케스트레이션한다. 각 에이전트는 자기 VM에서 도는 독립 세션이다 — [2차: fourweekmba](https://fourweekmba.com/ai-factory-and-cognition-ship-rival-tools-to-run-agent-fleets/) (1차 미확인)

**Factory (Droids) [공식이나 2차 경유]**
- **Environments**(재사용 가능한 sandbox로 몇 시간마다 재빌드됨) GA 2026-10-12. "Droid 1.0: Building a Frontier Harness"(2026-10-29)는 하네스를 모델 주변의 모든 것(프롬프트, 도구, 루프)으로 정의하고, 모델에 독립적이며 script-mode 프로그램형 tool calling을 쓴다고 주장한다. Factory 자체 수치이고 독립 검증은 없다 — [2차: fourweekmba](https://fourweekmba.com/ai-factory-and-cognition-ship-rival-tools-to-run-agent-fleets/), [2차: emergent.sh](https://emergent.sh/news/factory-droid-1-0-model-independent-coding)

**Amp (구 Sourcegraph) [2차]**
- Amp Inc.로 분사했다. "nearly any agent can get good results"라는 입장(wrapper보다 모델이 중요)이 인용된다 — [2차: fourweekmba](https://fourweekmba.com/ai-factory-and-cognition-ship-rival-tools-to-run-agent-fleets/) (1차 미확인)

### Inferences
- Stripe blueprint(결정적 노드 + 에이전트 노드)와 Devin Dynamic Workflows(결정적 스크립트가 에이전트를 오케스트레이션)는 같은 방향이다. 루프 제어 흐름은 코드가 소유하고 LLM은 노드 안에서만 돈다(12-factor F8과도 일치). loop-harness의 Go 백엔드에 "workflow DAG of deterministic/agentic steps" 모델을 두는 근거가 된다.
- "CI 재시도 상한(Stripe=2)", "사람만 merge(GitHub)", "egress 차단 sandbox"는 업계 공통 안전 기본값으로 볼 수 있다.

### Gaps
- Shopify, Vercel, Cloudflare, Meta, Uber, Airbnb의 에이전트 코딩 파이프라인 1차 포스트는 도구 호출 예산 안에서 확보하지 못했다(존재 여부 자체를 확인하지 못함).
- Factory "Building a Frontier Harness"와 Cognition Dynamic Workflows 원문은 직접 확인하지 못했다(2차 출처만 있음).
- Amp의 공식 harness 설계 글(예: subagent/oracle 등)은 확인하지 못했다.

## Q3. 하네스의 공통 해부 (Common anatomy)

### Takeaway
출처를 교차해 보면 하네스는 다음 구성요소로 수렴한다. instruction 파일 → spec/plan 산출물 → 작업 분해(한 루프에 한 항목) → 격리 sandbox → 결정적 검증 게이트(lint/type/test, shift-left) → inferential 리뷰 → 사람 리뷰/merge 권한 → 재시도/예산 상한 → 실패 모드 대응(reset, fresh context).

### Cited Findings
- **Instruction 파일**: AGENTS.md/CLAUDE.md/.cursorrules를 저장소에 커밋한다 — [Thoughtworks Radar](https://www.thoughtworks.com/radar/techniques/curated-shared-instructions-for-software-teams). 실수 기반으로 한 줄씩 늘려 간다 — [Hashimoto](https://mitchellh.com/writing/my-ai-adoption-journey). 디렉터리 범위로 지정하고 global은 최소화한다 — [Stripe](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- **Spec/Plan 산출물**: specs/ + fix_plan.md — [Huntley](https://ghuntley.com/ralph/). constitution/spec/plan/tasks — [Spec Kit](https://github.github.com/spec-kit/index.html). Research/Plan 문서와 단계별 compaction — [HumanLayer](https://humanlayer.dev/blog/advanced-context-engineering)
- **작업 분해**: 루프당 한 항목 — [Huntley](https://ghuntley.com/ralph/). planner→worker 단일 task — [Cursor](https://cursor.com/blog/scaling-agents). 작은 집중형 에이전트 — [12-factor](https://humanlayer.dev/blog/12-factor-agents). 계획 세션과 실행 세션 분리 — [Hashimoto](https://mitchellh.com/writing/my-ai-adoption-journey)
- **Sandbox/격리**: devbox(EC2 warm pool, egress 차단) — [Stripe](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2). ephemeral Actions 환경 + firewall — [GitHub](https://docs.github.com/copilot/concepts/coding-agent/about-copilot-coding-agent). 컨테이너, allowlist, 예산 제한 키 — [Willison](https://simonwillison.net/2025/Sep/30/designing-agentic-loops)
- **결정적 검증 게이트**: computational sensors를 매 변경마다 실행 — [Böckeler](https://martinfowler.com/articles/harness-engineering.html). backpressure(변경 단위 테스트, 타입체커) — [Huntley](https://ghuntley.com/ralph/). pre-push hook, lint 루프, CI 2라운드 — [Stripe](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2)
- **Inferential 검사/judge**: review agent, LLM-as-judge — [Böckeler](https://martinfowler.com/articles/harness-engineering.html). 사이클 경계의 judge — [Cursor](https://cursor.com/blog/scaling-agents)
- **Human-in-the-loop**: 에이전트에 merge/approve 권한 없음, 워크플로 실행에 사람 승인 필요 — [GitHub](https://docs.github.com/en/enterprise-cloud@latest/copilot/concepts/agents/cloud-agent/risks-and-mitigations). 사람이 100% 리뷰 — [Stripe](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2). tool 선택과 실행 사이 승인 지점 — [12-factor F8](https://humanlayer.dev/blog/12-factor-agents). read-only 백그라운드 에이전트 — [Hashimoto](https://mitchellh.com/writing/my-ai-adoption-journey)
- **비용/시간 예산**: CI 라운드 상한 — [Stripe](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents-part-2). 비용 키 한도 — [Willison](https://simonwillison.net/2025/Sep/30/designing-agentic-loops). 비싼 planner와 저렴한 worker의 혼합 — [2차: StartupHub](https://www.startuphub.ai/ai-news/technology/2026/ai-agent-swarms-rethink-model-economics)
- **실패 모드 대응**: placeholder, 중복 구현, 무한 루프, `git reset --hard` — [Huntley](https://ghuntley.com/ralph/). 위험 회피, lock 독점, drift에는 주기적 fresh start — [Cursor](https://cursor.com/blog/scaling-agents). 에러를 컨텍스트에 압축 — [12-factor F9](https://humanlayer.dev/blog/12-factor-agents)
- **Observability**: drift/health sensor, SLO 모니터링, 로그 이상 탐지 AI judge — [Böckeler](https://martinfowler.com/articles/harness-engineering.html)

### Inferences
- loop-harness의 최소 데이터 모델 후보: `Guide`(instruction/spec/skill), `Sensor`(computational|inferential, 실행 시점 = pre-commit|CI|continuous), `Plan/Task`, `Iteration`(새 컨텍스트, 한 항목), `Sandbox`, `Gate`(재시도 상한, 예산), `HumanCheckpoint`, `Trace`.
- 하네스 품질 지표(sensor 커버리지)가 업계에서 미해결 질문으로 남아 있다는 점은 차별화 기회가 될 수 있다.

### Gaps
- tracing과 observability 쪽 구체적 업계 표준(OpenTelemetry GenAI semantic conventions 등)은 이번 범위에서 조사하지 못했다.
- worktree 기반 병렬화의 회사 사례 1차 출처는 확보하지 못했다.

## Q4. 벤치마크와 하네스 평가 (SWE-bench Verified/Pro, Terminal-Bench, harness vs model)

### Takeaway
SWE-bench Verified는 오염과 포화 문제로 신뢰를 잃는 중이고 SWE-bench Pro로 옮겨가고 있다. Terminal-Bench 2.0 리더보드에서는 **같은 모델이라도 하네스에 따라 10~20%p 이상 차이**가 난다. 그래서 점수는 "모델+하네스" 조합의 점수로 읽어야 한다.

### Cited Findings
- **SWE-bench Pro** (Scale AI, 2025-09): 41개 저장소, 1,865개 인스턴스(공개 731, held-out 858, 상용 276)로 오염 저항형으로 설계됐다. 공개 당시 GPT-5는 23.3%, Claude Opus 4.1은 23.1%였다(Verified에서는 70% 이상) — [scale.com](https://scale.com/blog/swe-bench-pro), [arXiv 2509.16941](https://web3.arxiv.org/pdf/2509.16941)
- OpenAI가 Verified 점수 보고를 중단하고 Pro를 권장한다고 보도됐다. 어려운 과제의 59.4%에 결함이 있었다는 감사 결과가 근거로 인용된다 — [2차: tessl.io](https://tessl.io/blog/openai-moves-beyond-swe-bench-verified-as-coding-benchmarks-saturate/), [2차: aiweekly](https://aiweekly.co/node/5874)
- Pro의 grader가 시행의 약 1/3을 잘못 채점했다는 2026-05 Datacurve 감사가 보도됐으나 미확인이다 — [2차](https://liveinthefuture.org/stories/swe-bench-reality-gap)
- **Terminal-Bench 2.0**: GPT-5.3-Codex가 Terminus 2에서 64.7%, SageAgent에서 78.4%로 13.7%p 차이가 났다. 원인으로 terminal affordance, context packaging, 명령 실행 방식, 종료 기준이 꼽힌다 — [arXiv 2606.20683 survey](https://arxiv.org/pdf/2606.20683)
- 같은 Opus 4.6이 Claude Code에서 58.0%, Terminus-KIRA에서 74.7%라는 보고가 있다(2차 블로그) — [qepo17](https://qepo17.pages.dev/til/2026-04-02-terminal-bench-harness-gap), [danielvaughan](https://codex.danielvaughan.com/2026/04/09/harness-performance-terminal-bench/)
- 반론도 있다. Terminal-Bench 원 논문은 보통 더 강한 모델을 고르는 쪽이 scaffold를 바꾸는 것보다 효과가 크다고 본다 — [2차: emergentmind, arXiv 2601.11868](https://www.emergentmind.com/papers/2601.11868). 리더보드 순위의 운/실력 분석 — [LessWrong](https://www.lesswrong.com/posts/GiPmLmmbT6DyrwYkH/terminal-bench-leaderboard-rankings-luck-or-skill)
- Cursor는 "prompts matter more"(harness와 model보다)라고 했다 — [cursor.com](https://cursor.com/blog/scaling-agents)

### Inferences
- harness와 model의 기여 비중에 대해서는 출처들이 엇갈린다. 하네스 효과가 크다는 2차 블로그들과 모델 선택이 더 크다는 원 논문이 대립한다. 그래도 같은 모델에서 10%p 이상 차이가 날 수 있다는 점은 여러 출처가 일치한다.
- loop-harness 자체에도 작은 내부 eval 세트(자기 저장소 과제의 pass@k, 비용, 시간)를 두고 하네스 변경을 회귀 측정하는 기능이 업계 관행에 맞다.

### Gaps
- tbench.ai 공식 리더보드와 swebench.com 최신(2026-11) 수치를 직접 확인하지 못했다. 위 수치는 대부분 2차 출처다.
- Terminal-Bench 2.1의 존재와 변경점을 1차로 확인하지 못했다.
