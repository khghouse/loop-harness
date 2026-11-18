# Google의 에이전트 루프 / 하네스 설계 원칙 (조사 기준일 2026-11-18)

> 범위: Google Cloud / Google DeepMind / Google 엔지니어링이 공개한 에이전트 아키텍처, 에이전트 루프, 코딩 에이전트 하네스, 평가(AgentOps) 가이드.
> 주의: 2026년에 큰 변화가 2건 있음. (1) Gemini CLI는 2026-05-19 발표로 Antigravity CLI로 이관(무료/개인 사용자는 2026-06-18 서비스 종료). (2) ADK의 "template workflow agents"(Loop/Sequential/Parallel)는 문서상 deprecated 표기, graph/dynamic workflows로 대체 권장. A2A는 2026-08-17 Linux Foundation 산하 Agentic AI Foundation(AAIF) 프로젝트로 편입.

## Q1. Google 에이전트 백서(Agents / Agents Companion / Kaggle 5-day 과정)가 정의하는 에이전트 루프, 오케스트레이션 레이어, AgentOps/평가

### Takeaway
Google의 정본 프레이밍은 "에이전트 = 모델 + 도구 + 오케스트레이션 레이어(+ 배포/런타임)"이며, 오케스트레이션 레이어가 think-act-observe 루프(ReAct 등)를 돌리면서 계획·상태(메모리)·추론 전략을 관리한다. 2025-11 Kaggle 5부작 백서(Introduction to Agents → Tools/MCP → Context Engineering → Agent Quality → Prototype to Production)가 현재 기준 레퍼런스이고, 평가는 로그·트레이스·메트릭 관측성 + LLM-as-a-Judge + HITL 조합을 표준으로 삼는다.

### Cited Findings
- "Introduction to Agents" 백서: 저자 Alan Blount, Antonio Gulli, Shubham Saboo, Michael Zimmermann, Vladimir Vuskovic, 2025년 11월, Kaggle/Google 5-Day AI Agents Intensive의 Day 1 자료. 정본 링크 https://www.kaggle.com/whitepaper-introduction-to-agents — [HowAIWorks 요약](https://howaiworks.ai/kaggle-google-introduction-to-agents-whitepaper-2025/); [Kaggle](https://www.kaggle.com/whitepaper-introduction-to-agents)
- 오케스트레이션 레이어 = 에이전트의 운영 루프를 관리하는 통제 프로세스; 계획, 메모리(상태), 추론 전략 실행을 담당하고 CoT/ReAct 같은 프롬프팅 프레임워크를 사용. 'Think, Act, Observe' 루프를 실행 — [ppc.land](https://ppc.land/google-cloud-releases-comprehensive-agentic-ai-framework-guideline/); [techstrong.ai](https://techstrong.ai/features/the-orchestration-layer-why-your-ai-agent-needs-a-nervous-system/)
- 도구에는 HITL 기능 포함 가능: 워크플로를 일시정지하고 사용자 확인/입력을 요청 — [ppc.land](https://ppc.land/google-cloud-releases-comprehensive-agentic-ai-framework-guideline/)
- 2024년판 "Agents" 백서(구판)의 에이전트 정의: 목표 달성을 위해 세계를 관찰하고 보유 도구로 행동하는 애플리케이션; 오케스트레이션 레이어는 정보를 받아 내부 추론 후 다음 행동을 결정하는 순환 프로세스; 도구를 Extensions(에이전트 측 실행)와 Functions(클라이언트 측 실행)로 구분 — [AI Hero 리뷰](https://aihero.dev/google-agents-whitepaper-review) (구판, 2025-11 판으로 대체됨)
- 5부작 구성: Day1 Introduction to Agents(Agent Ops, identity/policy 제약, ADK로 간단한 에이전트), Day2 Agent Tools & Interoperability with MCP(안전한 도구 설계, 실행 전 human approval, long-running tool call), Day3 Context Engineering: Sessions & Memory(세션=단기 대화 이력, 메모리=장기 지식), Day4 Agent Quality(관측성 3요소 logs/traces/metrics, LLM-as-a-Judge, HITL, 장기 모니터링), Day5 Prototype to Production(배포 파이프라인, A2A, Vertex AI Agent Engine) — [Analytics Vidhya](https://www.analyticsvidhya.com/blog/2025/11/ai-agent-research-papers-by-google/)
- 과정은 2025-11-10~14 라이브 진행, 이후 Kaggle self-paced Learn Guide로 제공. Google은 2026-02-26 이 과정 기반 5개 개발자 가이드를 무상 공개 — [Kaggle Learn Guide](https://www.kaggle.com/learn-guide/5-day-agents); [@IT (atmarkit, 일본어)](https://atmarkit.itmedia.co.jp/ait/articles/2604/10/news057.html)
- 백서 페이지 수가 출처마다 다름(42p vs 54p) — [HowAIWorks](https://howaiworks.ai/blog/kaggle-google-introduction-to-agents-whitepaper-2025) vs [ppc.land](https://ppc.land/google-cloud-releases-comprehensive-agentic-ai-framework-guideline/)

### Inferences
- loop-harness 관점에서 Google 용어 매핑: 오케스트레이션 레이어 ≈ 하네스, think-act-observe ≈ 반복 루프 1회, AgentOps ≈ 관측성+평가+배포 파이프라인. "모델/도구/오케스트레이션/런타임" 4분할은 도구 비종속(tool-agnostic) 하네스의 아키텍처 경계로 그대로 차용 가능.
- Day4 Agent Quality의 "logs/traces/metrics + LLM judge + HITL"는 하네스 대시보드(Next.js)의 기본 관측 단위로 적합.

### Gaps
- Kaggle 백서 PDF 원문을 직접 추출하지 못함(Kaggle 페이지 JS 렌더링, PDF 텍스트 추출 도구 부재). 기억상(검증 안 됨): 2025-11판은 루프를 5단계("Get the mission → Scan the scene → Think it through → Take action → Observe and iterate")로, 에이전트 수준을 Level 0~4(핵심 추론 시스템 → 연결된 문제해결자 → 전략적 문제해결자 → 협업 멀티에이전트 → 자기진화 시스템)로 분류하고, Agent Quality 백서는 "outside-in" 평가와 "trajectory is the truth" 원칙을 제시한 것으로 알려짐. 인용 전 원문 확인 필요.
- 2025-02 "Agents Companion" 백서(AgentOps, multi-agent, agentic RAG 등) 원문은 이번 조사에서 확인하지 못함.

## Q2. Agent Development Kit (ADK): Loop/Sequential/Parallel 에이전트, 콜백, 평가, 세션/메모리, 컨텍스트 엔지니어링

### Takeaway
ADK는 결정론적 워크플로 에이전트(Sequential/Parallel/Loop)로 루프를 구성하고 종료는 반드시 `max_iterations` 또는 escalate(`exit_loop`)로 보장하라고 명시한다. 단, 2026년 현재 이 template workflows는 graph/dynamic workflows로 대체 권장(deprecated)이다. 컨텍스트는 "세션·메모리·아티팩트 위에 컴파일된 뷰"로 다루며, 평가는 최종 응답과 궤적(trajectory)을 둘 다 본다.

### Cited Findings
- ADK 문서 도메인이 google.github.io/adk-docs → adk.dev 로 이전(301) — [adk.dev](https://adk.dev/agents/workflow-agents/loop-agents/)
- LoopAgent: 하위 에이전트를 리스트 순서대로 반복 실행; 제어 흐름은 결정론적(모델이 루프 여부 결정 안 함). LoopAgent 자체는 종료를 결정하지 않으므로 무한 루프 방지 장치 필수 — (a) `max_iterations`(Python 기본값 None), (b) 하위 에이전트/콜백/도구가 `escalate=True` 설정, 내장 `exit_loop` 도구. 예시: SequentialAgent(writer 1회) → LoopAgent(critic → refiner, max_iterations=5), critic이 완료 문구를 내면 refiner가 exit_loop 호출. 상태 키(state)로 에이전트 간 출력 전달 — [ADK Loop agents](https://adk.dev/agents/workflow-agents/loop-agents/)
- 같은 페이지에 deprecation 공지: template workflows는 더 유연한 workflow 구조로 대체됨; Python/TS/Go/Kotlin에서는 graph-based workflows, dynamic workflows 권장. 사이트 배너: ADK TypeScript 2.0 GA(graph workflows 지원) — [ADK Loop agents](https://adk.dev/agents/workflow-agents/loop-agents/)
- 컨텍스트 엔지니어링(Google Developers Blog, Hangfei Lin, 2025-12-04): 핵심 명제 "Context is a compiled view over a richer stateful system." 원칙 3가지 — 저장과 표현 분리, 명시적 변환(이름 있는 순서형 processor 파이프라인), 기본은 최소 범위(scope by default; 더 필요하면 도구로 요청) — [Google Developers Blog](https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/)
- 계층 모델: Working context(호출마다 재구성·폐기), Session(구조화된 Event의 영속 로그), Memory(세션 간 검색 가능 지식), Artifacts(이름·버전 있는 대형 객체, 프롬프트에 붙이지 않고 핸들로 참조; LoadArtifactsTool로 필요 시 로드 후 오프로드) — 같은 출처
- Compaction: 임계치 도달 시 LLM이 슬라이딩 윈도로 오래된 이벤트를 요약해 이벤트로 다시 기록; 규칙 기반 필터 플러그인으로 결정론적 제거. 캐시 친화를 위해 안정적 내용(시스템 지시, 장기 요약)은 앞에, 동적 내용은 뒤에(prefix stability), "static instruction" 프리미티브 — 같은 출처
- 메모리 회상: reactive(에이전트가 도구로 검색) vs proactive(pre-processor가 유사도 검색 후 주입) — 같은 출처
- 멀티에이전트 컨텍스트: Agents-as-Tools(이력 없이 집중 프롬프트만) vs Agent Transfer(세션 뷰를 넘겨 이어서 수행); `include_contents`="none"이면 이전 이력 미전달; 핸드오프 시 이전 assistant 메시지를 서술형 컨텍스트로 재구성하고 도구 호출을 실행한 에이전트에 귀속시켜 새 에이전트가 자기 행동으로 오인하지 않게 함 — 같은 출처
- ADK 평가: LLM 변동성 때문에 결정론적 pass/fail 부적합 → 최종 응답 + 궤적(도구 호출 시퀀스) 모두 평가. 단위테스트형 `.test.json`(세션 1개) vs 통합테스트형 evalset(다중 세션, user simulation 시나리오). 메트릭: `tool_trajectory_avg_score`(정확 일치), `response_match_score`(ROUGE-1), `final_response_match_v2`(LLM judge), 루브릭 기반(응답 품질/도구 사용/멀티턴 궤적), `hallucinations_v1`, `safety_v1`, 효율 지표(도구 호출 수, 추론 호출 수, 토큰, 소요시간; pass/fail 아님). 기본 임계치: trajectory 1.0, response 0.8. 실행 경로: `adk web`(Eval/Trace 탭), pytest(`AgentEvaluator.evaluate`), `adk eval` CLI(CI용), `adk conformance`(기준 상호작용 기록 후 replay로 회귀 탐지) — [ADK Evaluate](https://adk.dev/evaluate/)
- 권장 실천: CI/회귀는 trajectory+response match로 빠르고 예측가능하게; 참조답 있으면 final_response_match_v2; 참조 없으면 루브릭; 프로덕션은 hallucination/safety/multi-turn 추가; 효율 지표를 추적해 품질 점수가 놓치는 비용 증가 감지 — [ADK Evaluate](https://adk.dev/evaluate/)

### Inferences
- loop-harness의 루프 정의에 그대로 이식 가능한 패턴: "generator → critic → refiner + 명시적 종료 조건(max_iterations OR escalate 신호)". 하네스가 모델에게 종료 판단을 맡기지 않고 결정론적 컨트롤러가 반복을 통제한다는 점이 Google의 설계 입장.
- 2026 시점 표준은 고정 Loop/Sequential 템플릿보다 그래프 워크플로 쪽으로 이동 중 → loop-harness의 Go 백엔드는 루프를 "그래프의 사이클 + 종료 엣지"로 모델링하는 것이 ADK 2.x와 정합.
- 세션 = append-only 이벤트 로그, working context = 매 호출 컴파일 결과, 아티팩트 = 핸들 참조 — 이 3분할은 도구 비종속 하네스의 저장 계층 설계에 직접 적용 가능.

### Gaps
- ADK 콜백(before/after_agent, before/after_model, before/after_tool) 문서와 graph workflow 문서 원문은 이번에 직접 확인하지 못함(adk.dev 해당 페이지 미조회).
- ADK 2.0의 정확한 GA 일자(Python)와 LoopAgent → graph workflow 매핑 방식 미확인.

## Q3. Gemini CLI / Jules / Antigravity: 하네스 구조

### Takeaway
Gemini CLI는 GEMINI.md 계층형 컨텍스트, 확장(extensions), MCP, 체크포인팅, 샌드박스, 헤드리스 모드, 승인 모드(default/auto_edit/plan/yolo), 훅, Agent Skills, 서브에이전트를 갖춘 사실상 "레퍼런스 하네스"였으나, 2026-05-19 Antigravity CLI(Go 기반, Antigravity 2.0과 동일한 서버측 하네스 공유)로 이관이 발표되었다. Skills/Hooks/Subagents/Extensions(→ plugins)는 이관 대상 기능으로 명시되었다.

### Cited Findings
- 이관 공지(2026-05-19, Dmitry Lyalin·Taylor Mullen): Gemini CLI를 Google Antigravity의 터미널 경험인 Antigravity CLI로 통합; Go로 작성, 다수 에이전트를 백그라운드 비동기 실행; 초기엔 "1:1 feature parity" 아님. 2026-06-18 Google AI Pro/Ultra 및 무료 Code Assist 개인 사용자 대상 Gemini CLI·Code Assist IDE 확장 서비스 중단. Code Assist Standard/Enterprise와 유료 API 키 사용자는 유지. 이관 기능: Agent Skills, Hooks, Subagents, Extensions(Antigravity plugins로). "powerful server-side harness", Antigravity 2.0과 "shares the same agent harness" — [Google Developers Blog](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)
- 비판적 반응(오픈소스 사용자 대상 bait-and-switch 프레이밍) 존재 — [The Register](https://www.theregister.com/ai-ml/2026/05/20/bye-bye-gemini-cli-google-nudges-devs-toward-antigravity/5243605) (의견)
- Gemini CLI 리포: Apache 2.0; 현재 README는 Code Assist Standard/Enterprise용으로 유지, Gemini 4+에 최적화 안 됨 명시. 기능: GEMINI.md 지속 컨텍스트, 커스텀 extensions·slash commands, MCP(`~/.gemini/settings.json`), 체크포인팅, 샌드박스 + trusted folders, 헤드리스(text/JSON/stream-JSON 출력), 내장 파일/셸/웹fetch 도구, 토큰 캐싱, 텔레메트리. 릴리스: preview/stable 주간, nightly 일간 — [GitHub google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)
- 설정 레퍼런스: `context.fileName`은 문자열 또는 배열(예: `["CONTEXT.md","GEMINI.md"]`) → AGENTS.md 지정 가능(문서 예시에는 AGENTS.md 직접 언급 없음); `context.memoryBoundaryMarkers`(기본 `[".git"]`)에서 상향 탐색 중단; `general.checkpointing.enabled`(기본 false); `tools.sandbox`(docker/podman/lxc/windows-native), `tools.sandboxNetworkAccess`(기본 false), macOS `SEATBELT_PROFILE`; `general.defaultApprovalMode` = default/auto_edit/plan, YOLO는 CLI 플래그로만 가능하고 관리자가 `security.disableYoloMode`로 차단 가능; hooks 이벤트 `BeforeTool, AfterTool, BeforeAgent, AfterAgent, Notification, SessionStart, SessionEnd, PreCompress, BeforeModel, AfterModel, BeforeToolSelection`; `skills.enabled`(기본 true); `experimental.enableAgents`(로컬/원격 서브에이전트), `experimental.adk.agentSessionSubagentEnabled`(서브에이전트를 ADK AgentSession 프로토콜로 라우팅) — [geminicli.com Configuration](https://geminicli.com/docs/reference/configuration)
- Plan mode(읽기 전용으로 복잡한 변경 계획)와 Subagents는 공식 문서 인덱스에서 실험 기능으로 표시 — [geminicli.com docs](https://geminicli.com/docs) (검색 스니펫 경유)
- Jules: 비동기 클라우드 코딩 에이전트. 2025-10 Jules Tools CLI와 공개 API 출시; 웹에서 계획 검토 후 "Approve plan"으로 구현 시작, API에는 `requirePlanApproval` 옵션; AGENTS.md를 프로젝트 컨텍스트로 읽는다고 보고됨 — [Slashdot](https://developers.slashdot.org/story/25/10/03/2140223/googles-jules-enters-developers-toolchains-as-ai-coding-agent-competition-heats-up); [Machine Learning Mastery](https://machinelearningmastery.com/practical-agentic-coding-with-google-jules/); [rywalker.com 분석](https://rywalker.com/research/jules) (모두 2차 출처)

### Inferences
- Gemini CLI 훅 이벤트 집합(Before/After Tool·Agent·Model, SessionStart/End, PreCompress)은 Claude Code 훅과 거의 대응 → 도구 비종속 하네스의 공통 라이프사이클 이벤트 스키마를 정의할 때 교집합 기준으로 사용 가능.
- "approval mode = default / auto_edit / plan / yolo"는 하네스 권한 단계의 업계 공통 축.
- Google의 전략 축이 "로컬 오픈소스 CLI" → "서버측 공유 하네스 + 다중 비동기 에이전트(Antigravity)"로 이동. loop-harness가 Gemini 계열을 지원하려면 Gemini CLI(엔터프라이즈 잔존)와 Antigravity CLI 양쪽 어댑터를 고려해야 함.

### Gaps
- Antigravity CLI의 오픈소스 여부/라이선스, 명령어 이름, AGENTS.md 지원 여부를 1차 출처로 확인하지 못함(공지에 언급 없음).
- Jules의 AGENTS.md 지원, 계획 승인 자동화 세부는 Google 공식 문서(jules.google/docs)로 확인하지 못함.
- Gemini CLI 공식 문서가 AGENTS.md를 기본값으로 읽는지 여부는 미확인(설정으로 지정은 가능).

## Q4. Google의 개방형 프로토콜: A2A, MCP와의 관계, Linux Foundation 거버넌스, AGENTS.md / Agent Skills 참여

### Takeaway
A2A는 Google이 2025-04 발표한 에이전트 간 통신 표준으로, 2025년 Linux Foundation에 기증되었고 v1.0(2026-03)을 거쳐 2026-08-17 MCP·AGENTS.md·goose와 함께 AAIF 호스티드 프로젝트가 되었다. Google의 공식 포지션은 "MCP = 에이전트↔도구, A2A = 에이전트↔에이전트, 둘 다 사용"이다.

### Cited Findings
- A2A: "opaque agentic applications" 간 통신·상호운용 표준; 에이전트가 내부 메모리·도구·로직을 공유하지 않고 협업(opacity); MCP는 도구/데이터 연결, A2A는 에이전트 간 연결이며 둘 다 쓰라고 권장; Apache 2.0; 모든 공식 SDK가 A2A v1.0 스펙 구현 — [a2a-protocol.org](https://a2a-protocol.org/latest/)
- AAIF 공지(2026-08-17): A2A가 AAIF 호스티드 프로젝트로 편입. AAIF 프로젝트: AGENTS.md(에이전트에 대한 규약/지시), goose(런타임), MCP(도구 연결), agentgateway(라우팅·정책·관측성). A2A v1.0(첫 안정 스펙)은 2026-03 출시: multi-protocol bindings, 버전 협상, 멀티테넌시, 서명된 agent card. Google은 2025-04 A2A 발표, AWS·Cisco·Microsoft·Salesforce·SAP·ServiceNow 등과 함께 LF에 기증. Google Cloud는 ADK, Agent Engine, Cloud Run, GKE로 A2A 에이전트 지원 — [AAIF Blog](https://aaif.io/blog/a2a-joins-aaif)
- AAIF는 2025-12 Anthropic MCP, Block goose, OpenAI AGENTS.md를 창립 기여로 출범 — [FlowHunt](https://www.flowhunt.io/blog/agentic-ai-foundation-a2a-mcp-standards/) (2차)
- Google 측 AGENTS.md/Agent Skills 참여 신호: Gemini CLI가 Agent Skills 지원(기본 활성), `context.fileName`으로 AGENTS.md 지정 가능; Antigravity CLI로 Agent Skills 이관 명시 — [geminicli.com Configuration](https://geminicli.com/docs/reference/configuration); [Google Developers Blog](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)

### Inferences
- 2026-11 시점 업계 "표준 스택"은 AAIF 기준 MCP(도구) + A2A(에이전트 간) + AGENTS.md(리포 지시)로 수렴. loop-harness는 에이전트 지시 파일로 AGENTS.md를 1차로 채택하고 GEMINI.md/CLAUDE.md는 어댑터로 심볼릭/생성하는 방식이 표준 정합적.
- 하네스가 외부 에이전트(Codex, Claude Code 등)를 "원격 워커"로 다루는 확장을 원하면 A2A의 Task/Agent Card 모델이 중립 인터페이스 후보.

### Gaps
- A2A 핵심 개념(Agent Card, Task 수명주기 상태, Message/Part/Artifact, JSON-RPC/gRPC/REST 바인딩) 스펙 원문은 이번에 직접 조회하지 못함.
- Google이 AAIF 회원사인지, Agent Skills 표준화(agentskills.io 등)에 공식 참여하는지 1차 출처 미확인.

## Q5. 검증 루프 관련 Google 엔지니어링 실천 (AI 코드 마이그레이션 등)

### Takeaway
Google의 대규모 AI 코드 마이그레이션 경험 보고서는 "결정론적 탐색(Code Search/Kythe/AST) → LLM 편집(병렬 다중 후보) → 빌드·테스트 자동 검증 → 실패 시 에러를 넣어 LLM 수리 → 최선 후보 선택 → 사람 리뷰·점진 롤아웃"을 표준 루프로 제시한다. 성공 지표는 AI 작성 코드 비율이 아니라 end-to-end 소요시간 단축이다.

### Cited Findings
- 논문: "How is Google using AI for internal code migrations?" arXiv:2501.06972, 2025-01-12, Stoyan Nikolov, Daniele Codecasa, Anna Sjövall, Maxim Tabachnyk, Satish Chandra(Google Core), Siddharth Taneja, Celal Ziftci(Google Ads). 연구가 아닌 경험 보고서 — [arXiv](https://arxiv.org/html/2501.06972v1)
- 워크플로: (1) 전문가가 시드 지정 후 Code Search·Kythe 교차참조·스크립트로 후보 파일 상위집합 탐색, 정규식/파싱 필터로 오탐 제거, 관련 테스트·인터페이스 파일 확장; (2) 내부 코드로 파인튜닝된 Gemini가 diff 생성, 관련 파일은 한 프롬프트로 묶어 일관성 유지, 프롬프트 변형을 병렬 실행(pass@k 유사); (3) 빌드+단위테스트로 자동 검증, 실패 시 변경 파일과 에러로 LLM 수리 시도, 후보를 검증 결과와 예시 diff 대비 점수화해 최선 선택; (4) 엔지니어 검토 후 샤드로 분할해 코드 오너에게 리뷰, 랜딩/롤아웃은 사람이 주도 — [arXiv](https://arxiv.org/html/2501.06972v1)
- 수치: 성공 기준은 end-to-end 시간 50% 이상 단축. int32→int64(Ads) 랜딩 코드 약 80%가 완전 AI 작성, 총 시간 약 50% 단축 추정. JUnit3→4: 3개월간 5,359 파일 / 149,000+ 라인, AI 생성 코드 약 87%가 수정 없이 커밋, 병목은 리뷰어 용량. Joda Time→java.time: 소규모 클러스터에서 약 89% 시간 절감 추정 — [arXiv](https://arxiv.org/html/2501.06972v1)
- 교훈: 프롬프팅만으론 불충분, AST·휴리스틱·LLM 결합이 더 나음(AST 검사가 원치 않는 편집을 싸고 예측가능하게 잡음); 작고 잘 정의된 LLM 하위작업이 튜닝·재사용 쉬움; 검증·수리 루프가 불완전한 1차 시도를 다수 회복; 사람 리뷰와 점진 롤아웃이 병목이므로 변경 속도 제한 필요; AI 코드 비율보다 end-to-end 시간 절감이 더 나은 지표 — [arXiv](https://arxiv.org/html/2501.06972v1)

### Inferences
- loop-harness의 "검증 루프" 기본형으로 그대로 채택 가능: 결정론적 게이트(빌드/테스트/린트/AST 체크)가 루프 종료 조건이며, 실패 출력(에러 로그)을 다음 반복 입력으로 주입. ADK LoopAgent의 critic→refiner+escalate와 동일 구조이며, critic을 LLM 대신 테스트 러너로 두는 것이 Google 실천과 정합.
- 다중 후보 병렬 생성 후 검증 점수로 선택(best-of-k)은 하네스의 "parallel worktree" 기능의 근거가 됨.
- 리뷰가 병목이라는 교훈 → 하네스에 변경 샤딩, 리뷰 큐, 처리량 제한 기능이 가치 있음.

### Gaps
- Google DeepMind/Google Research의 별도 "test-driven agent workflow" 또는 SWE 에이전트 하네스 설계 문서(예: 내부 코딩 에이전트 논문)는 이번 조사에서 추가로 찾지 못함.
- 2025-2026년 후속 Google 마이그레이션/에이전트 SWE 논문(예: 후속 arXiv 보고서) 존재 여부 미확인.
