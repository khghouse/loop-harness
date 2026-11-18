# Anthropic이 공개한 에이전트 루프 / 하네스 설계 원칙 (2026-11-18 기준)

> 범위: Anthropic 1차 자료(anthropic.com/engineering, claude.com/blog, code.claude.com/docs, agentskills.io)만 우선 사용. 각 항목에 게시일 표기. 공식 engineering 목록 페이지 인덱스: [Anthropic Engineering](https://www.anthropic.com/engineering) (2026-11 기준 29개 글, 그중 에이전트·하네스 관련 글은 아래에서 다룸).
> 시간 순서 핵심 문헌:
> - 2024-12-19 Building effective agents
> - 2025-06-13 How we built our multi-agent research system
> - 2025-09-11 Writing effective tools for agents — with agents
> - 2025-09-29 Effective context engineering for AI agents / Building agents with the Claude Agent SDK
> - 2025-10-16 Equipping agents for the real world with Agent Skills (2025-12-18 오픈 표준화 업데이트)
> - 2025-10-20 Beyond permission prompts (Claude Code sandboxing)
> - 2025-11-04 Code execution with MCP
> - 2025-11-24 Advanced tool use
> - 2025-11-26 Effective harnesses for long-running agents
> - 2025-12-09 MCP를 Agentic AI Foundation에 기증
> - 2026-01-09 Demystifying evals for AI agents
> - 2026-02-05 Building a C compiler with a team of parallel Claudes / Quantifying infrastructure noise in agentic coding evals
> - 2026-03-24 Harness design for long-running application development
> - 2026-03-25 How we built Claude Code auto mode
> - 2026-04-08 Scaling Managed Agents: Decoupling the brain from the hands
> - 2026-05-25 How we contain Claude across products
> - 2026-11-06 The components of agent memory
> - 2026-11-13 Building strong safety evaluations
>
> 출처: [Anthropic Engineering index](https://www.anthropic.com/engineering)

## Q1. "Building effective agents"의 핵심 (workflows vs agents, augmented LLM, evaluator-optimizer 등)

### Takeaway
2024-12-19에 나온 이 글은 지금도 Anthropic 에이전트 설계의 기초 어휘입니다. 정의는 "agent = 환경 피드백(ground truth)을 받아 루프 안에서 도구를 쓰는 LLM"입니다. 가장 단순한 해법에서 시작하고, 정해진 코드 경로(workflow)와 모델이 스스로 흐름을 정하는 구조(agent)를 구분하며, 도구 인터페이스(ACI)에 프롬프트만큼 공을 들이라고 합니다. 현재 페이지에는 "도구 환경이 많이 바뀌었다"는 안내와 함께 Managed Agents로 가는 포인터가 붙어 있습니다.

### Cited Findings
- 게시일 2024-12-19, 저자 Erik S. 와 Barry Zhang. 현재 페이지에 "이후 툴링 지형이 많이 바뀌었다"는 안내와 Managed Agents 포인터가 추가돼 있음 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- **Workflows**는 LLM과 도구를 미리 정한 코드 경로로 오케스트레이션하고, **Agents**는 LLM이 자기 프로세스와 도구 사용을 동적으로 결정함. 둘을 묶어 "agentic systems"라고 부름 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- 가장 단순한 해법부터 시작할 것. agentic 시스템은 지연·비용을 성능과 맞바꾸므로 복잡성은 성능 향상이 입증될 때만 추가함. 대부분의 경우 단일 LLM 호출에 retrieval과 in-context 예시를 더하는 것으로 충분함 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- **Augmented LLM**(retrieval + tools + memory)이 기본 블록이고, 그 통합 방법의 하나로 MCP를 언급함 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- 5가지 워크플로 패턴 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
  - **Prompt chaining**: 순차 단계 사이에 프로그램적 "gate" 체크를 둠
  - **Routing**: 입력을 분류해 전문 경로로 보냄(쉬운 질문은 작은 모델로)
  - **Parallelization**: 독립 하위작업을 나누는 sectioning과 같은 작업을 여러 번 돌리는 voting
  - **Orchestrator-workers**: 중앙 LLM이 하위작업을 동적으로 분해·위임·종합함. 여러 파일을 수정하는 코딩 제품이 예시
  - **Evaluator-optimizer**: 생성 LLM과 평가 LLM이 피드백 루프를 돎. 평가 기준이 명확하고 반복이 측정 가능한 가치를 낼 때 적합
- 자율 에이전트는 각 단계에서 환경으로부터 "ground truth"(도구 결과, 코드 실행 결과)를 얻어 진척을 판단해야 함. 종료 조건으로 "최대 반복 횟수" 같은 stopping condition을 권장하고, 체크포인트나 막혔을 때는 사람 피드백을 받도록 일시정지함 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- 자율성은 비용과 오류 누적 위험을 키우므로 샌드박스에서 충분히 테스트하고 가드레일을 둘 것 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- 3대 원칙 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
  1. 단순성
  2. 투명성: 에이전트의 계획 단계를 명시적으로 보여줌
  3. ACI(agent-computer interface)를 문서화·테스트로 정교하게 설계
- 코딩 에이전트가 특히 유효한 이유는 자동 테스트로 검증할 수 있고, 테스트 피드백으로 반복할 수 있으며, 품질을 측정할 수 있기 때문. 단, 사람 리뷰는 여전히 필수 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
- ACI(도구 프롬프트) 지침 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
  - 모델 입장에서 생각할 것
  - 주니어 개발자용 docstring처럼 예시·경계·유사 도구와의 차이를 적을 것
  - 다양한 입력으로 테스트할 것
  - poka-yoke(실수하기 어렵게) 설계를 할 것
  - SWE-bench 에이전트에서는 "전체 프롬프트보다 도구 최적화에 더 많은 시간을 썼다". 절대경로 강제로 오류를 제거함
- 프레임워크는 프롬프트와 응답을 가리는 추상화를 더할 수 있음. LLM API를 직접 쓰는 것부터 시작하고, 프레임워크를 쓴다면 내부 코드를 이해할 것 — [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

### Inferences
- loop-harness의 기본 루프 모델은 "LLM + 도구 + 환경 피드백 + 종료조건(max iterations/budget)"이면 이 글의 정의와 정확히 일치합니다. evaluator-optimizer는 "생성 에이전트 + 검증자" 루프의 업계 공용 명칭으로 쓰기 좋습니다.
- "프레임워크 추상화 최소화"는 도구 비종속(tool-agnostic) 하네스가 에이전트 CLI를 얇게 감싸는 설계를 뒷받침합니다.

### Gaps
- 페이지에 붙은 "툴링 변화" 안내의 정확한 추가 날짜는 확인하지 못했습니다.

## Q2. 장기 실행 에이전트 / 하네스에 관한 엔지니어링 글들

### Takeaway
Anthropic의 하네스 설계는 다음 순서로 발전했습니다.
1. 2025-11: initializer와 coding 에이전트를 나누고 feature list, progress 파일, git으로 상태를 넘김
2. 2026-03: planner, generator, evaluator 3-에이전트 구조와 sprint contract 도입
3. 2026-04: 세션 로그(brain/hands 분리)를 영속 상태로 두는 Managed Agents

일관된 메타원칙은 "하네스의 모든 구성요소는 모델이 못하는 것에 대한 가정이며, 모델이 좋아지면 낡는다"입니다. 따라서 가정을 주기적으로 재검증하고 하나씩 제거해 봐야 합니다.

### Cited Findings
**Effective harnesses for long-running agents (2025-11-26, Justin Young)**
- 문제: 세션마다 기억이 없음("인수인계 노트 없는 교대근무"). compaction만으로는 부족했고, Opus 4.5를 Agent SDK 루프로 돌려도 고수준 프롬프트 하나로 프로덕션급 웹앱을 만들지 못함 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- 실패 패턴 두 가지 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
  - 한 번에 너무 많이 하다 컨텍스트가 바닥나 반쯤 구현된 상태로 끝남
  - 이전 진척을 보고 너무 일찍 "완료"를 선언함
- 해법 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
  - **Initializer agent**(첫 세션)가 `init.sh`, `claude-progress.txt`, feature list JSON, 첫 git commit을 만듦
  - **Coding agent**(이후 세션)는 한 번에 한 기능만 구현하고 "clean state"(main에 머지 가능한 상태)로 끝냄
  - 두 에이전트는 초기 프롬프트만 다르고 시스템 프롬프트·도구·하네스는 동일함
- Feature list — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
  - 200개 이상 기능을 모두 failing으로 시작함
  - 각 항목은 category, description, test steps, `passes` 필드를 가지며, 에이전트는 `passes`만 수정할 수 있음
  - Markdown이 아니라 JSON을 쓴 이유는 모델이 부적절하게 고칠 가능성이 낮아서임
  - "It is unacceptable to remove or edit tests" 같은 강한 지시를 둠
- 세션 시작 루틴 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
  1. `pwd` 확인
  2. git log와 progress 파일 읽기
  3. feature list에서 최우선 미완료 기능 선택
  4. `init.sh`로 dev 서버를 띄우고 기본 E2E 테스트를 돌려, 새 작업 전에 깨진 상태를 먼저 발견
  - 세션 끝에는 커밋하고 progress를 갱신함
- 검증: unit test나 curl만으로 완료를 표시하는 경향이 있었음. 브라우저 자동화(Puppeteer MCP)로 "사람 사용자처럼" 테스트하게 하자 크게 개선됨. 단, 브라우저 네이티브 alert 모달은 볼 수 없는 한계가 있음 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- 남은 질문: 범용 에이전트 하나가 나은지, 테스트·QA·정리용 멀티에이전트가 나은지 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

**Harness design for long-running application development (2026-03-24, Prithvi Rajasekaran, Anthropic Labs)**
- 실패 원인 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
  - **coherence 손실**: 일부 모델은 "context anxiety" 때문에 컨텍스트 한계가 가깝다고 느끼면 일을 서둘러 마무리함
  - **자기평가 편향**: "agents reliably skew positive when grading their own work"
- Context reset과 compaction 비교 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
  - reset은 구조화된 handoff로 새 에이전트를 시작하는 방식임
  - Sonnet 4.5에서는 reset이 필수였고, Opus 4.5에서는 그 행동이 대부분 사라져 자동 compaction만 씀
- Frontend 실험: GAN에서 영감받은 generator와 evaluator — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
  - 평가 기준 4개(design quality, originality, craft, functionality)를 두고, few-shot 점수 예시로 evaluator를 보정함
  - evaluator는 Playwright MCP로 실제 페이지를 조작하고 채점함
  - 5~15회 반복함
- 3-에이전트 구조 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
  - **Planner**: 1~4문장 프롬프트를 제품 스펙으로 확장함. 기술 세부는 피해 오류가 연쇄되지 않게 함
  - **Generator**: sprint 단위로 구현하고 git을 씀
  - **Evaluator**: Playwright로 UI·API·DB 상태를 검사함. 기준별 hard threshold를 하나라도 못 넘으면 sprint 실패
- **Sprint contract**: 각 sprint 전에 generator가 "무엇을 만들고 어떻게 검증할지"를 제안하고, evaluator와 합의할 때까지 반복함. 통신은 파일로 함 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- 비용 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
  - 레트로 게임 메이커: solo는 20분·$9, full harness는 6시간·$200. solo 결과물은 핵심 기능이 깨져 있었음
  - DAW(Opus 4.6, 단순화한 V2): 3시간 50분·$124.70
- Opus 4.6으로 단순화한 내용 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
  - sprint 구조를 제거함
  - planner는 유지함(없으면 범위가 과소해짐)
  - evaluator는 마지막 1회 실행하는 조건부 단계로 바꿈
  - 처음에 급격히 단순화했다가 실패한 뒤, 구성요소를 하나씩 제거해 어떤 것이 load-bearing인지 확인함
- 인용: "every component in a harness encodes an assumption about what the model can't do on its own" / "Out of the box, Claude is a poor QA agent." / evaluator는 "worth the cost when the task sits beyond what the current model does reliably solo" — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)

**Scaling Managed Agents: Decoupling the brain from the hands (2026-04-08, Lance Martin, Gabe Cemaj, Michael Cohen)**
- 세 컴포넌트를 OS처럼 가상화함. 각각 독립적으로 교체·실패할 수 있음 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
  - **Session**: append-only 이벤트 로그
  - **Harness**: Claude를 호출하고 도구 호출을 라우팅하는 루프
  - **Sandbox**: 코드 실행 환경
- 처음에는 하나의 컨테이너에 다 넣었는데, 이 컨테이너가 "pet"이 되어 죽으면 세션이 유실되고 디버깅도 어려웠음 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- Session은 컨텍스트 창 바깥의 영속 상태임 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
  - compaction이나 trimming은 비가역적이므로 모든 것을 세션에 저장함
  - `getEvents()`로 위치 기반 slice를 읽어 재개·되감기를 할 수 있음
  - 하네스는 이벤트를 컨텍스트에 넣기 전에 변환할 수 있음(캐시 적중률 개선 등)
- Harness는 stateless임. `wake(sessionId)`, `getSession(id)`, `emitEvent(id, event)`로 크래시 후 마지막 이벤트부터 재개함 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- Sandbox는 단일 인터페이스 `execute(name, input) → string`을 가진 도구 하나로 다룸 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
  - 컨테이너는 "cattle"로 취급함. 실패하면 도구 오류로 Claude에게 돌려주고 `provision({resources})`로 새로 만듦
- 자격증명은 샌드박스에 두지 않음. 프롬프트 인젝션 대비책임 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
  - git 토큰은 setup 시점에 remote 설정에만 씀
  - MCP OAuth 토큰은 vault와 프록시로 처리함
- "meta-harness": 인터페이스 모양에는 의견을 갖되, 그 뒤에서 무엇이 돌아가는지(Claude Code든 작업별 하네스든)에는 의견을 갖지 않음 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- 디커플링 효과: TTFT p50이 약 60%, p95가 90% 이상 감소 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- 인용: context reset은 Opus 4.5에서 "dead weight"가 됨 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)

**Building agents with the Claude Agent SDK (2025-09-29, Thariq Shihipar)**
- Claude Code SDK를 Claude Agent SDK로 개명함. Claude Code의 하네스가 코딩 외(리서치, 노트 등) 에이전트에도 범용으로 쓰인다는 이유. 핵심 원칙은 "give your agents a computer" — [Agent SDK blog](https://claude.com/blog/building-agents-with-the-claude-agent-sdk) (원래 anthropic.com/engineering URL은 이 주소로 308 리다이렉트됨)
- **에이전트 루프: gather context → take action → verify work → repeat** — [Agent SDK blog](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
  - Gather context
    - agentic search(grep/tail 등 bash)를 먼저 쓸 것. semantic search는 빠르지만 덜 정확하고 유지보수와 투명성이 떨어짐
    - 폴더 구조 자체가 context engineering임
    - 병렬·격리된 subagent를 씀
    - compaction(`/compact` 기반)을 씀
  - Take action
    - 도구는 컨텍스트에서 눈에 띄는 자리를 차지하므로 핵심 행동만 둠
    - bash와 스크립트를 활용함
    - 코드 생성: 정밀하고 조합·재사용이 가능함
    - MCP로 인증·외부 통합을 처리함
  - Verify work
    - 규칙 기반 피드백: lint가 대표적이고, JS보다 TS를 권장함(피드백 층이 더 많아서)
    - 시각 피드백: 스크린샷, Playwright MCP
    - LLM-as-judge: 견고하지 않고 비용과 지연이 늘어나므로 성능 이득이 그만한 가치가 있을 때만 씀
- 개선 방법 — [Agent SDK blog](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
  - 실패를 정밀하게 관찰할 것
  - 정보를 못 찾으면 검색 API 구조를 바꿀 것
  - 반복되는 실패에는 도구 호출에 형식 규칙을 추가할 것
  - 스스로 고치지 못하면 대체 도구를 줄 것
  - 실사용 기반 대표 eval 세트를 구축할 것

**How we built our multi-agent research system (2025-06-13)**
- Orchestrator-worker 구조: lead agent가 계획하고 subagent를 병렬로 생성함. 별도의 CitationAgent가 출처를 귀속함 — [Multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system)
- 성능 수치 — [Multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system)
  - Opus 4 lead + Sonnet 4 subagent 구성이 단일 Opus 4보다 내부 eval에서 90.2% 우수
  - BrowseComp에서 성능 분산의 약 80%를 토큰 사용량이 설명함
  - 에이전트는 채팅의 약 4배, 멀티에이전트는 약 15배 토큰을 씀
- 멀티에이전트는 넓고 병렬화 가능한 작업에 적합하고, 공유 컨텍스트가 많이 필요한 작업(대부분의 코딩)에는 부적합함 — [Multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system)
- 프롬프트 원칙 — [Multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system)
  - 에이전트를 단계별로 시뮬레이션해 볼 것
  - 위임할 때 목표, 출력 형식, 도구 지침, 경계를 명시할 것
  - 복잡도에 따라 노력 규모를 조절할 것
  - Claude가 도구 설명을 고치게 하면 작업시간이 40% 단축됨
  - 넓게 검색한 뒤 좁힐 것
  - 병렬 도구 호출로 최대 90% 단축됨
- 평가: 약 20개 쿼리로 시작하고, LLM judge 루브릭과 사람 테스트를 함께 씀 — [Multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system)
- 프로덕션 운영 — [Multi-agent research](https://www.anthropic.com/engineering/multi-agent-research-system)
  - 오류가 누적되므로 체크포인트에서 재개하고 retry함
  - 전체 tracing을 둠
  - rainbow deployment를 씀
  - 계획을 외부 메모리에 저장함
  - 완료된 단계를 요약하고 "fresh subagents with clean contexts"로 이어감

**Writing effective tools for agents — with agents (2025-09-11)**
- 프로세스 — [Writing tools](https://www.anthropic.com/engineering/writing-tools-for-agents)
  1. 프로토타입(로컬 MCP 서버를 Claude Code에 연결)
  2. 검증 가능한 결과가 있는 현실적 다단계 eval
  3. transcript 분석
  4. Claude Code에 transcript를 붙여 넣어 도구 리팩터링, held-out 세트로 과적합 방지
- 원칙 — [Writing tools](https://www.anthropic.com/engineering/writing-tools-for-agents)
  - API를 그대로 감싸지 말고 고영향 워크플로 단위로 통합할 것
  - namespacing(예: `asana_projects_search`)
  - UUID 대신 사람이 읽을 수 있는 필드를 반환할 것
  - `response_format` enum(concise/detailed). concise는 토큰이 약 1/3
  - pagination, filter, truncation. Claude Code의 도구 응답 기본 상한은 25,000 토큰
  - 행동으로 옮길 수 있는(actionable) 오류 메시지
  - 신입에게 설명하듯 쓴 설명과 명확한 파라미터명(`user_id`)
  - "More tools don't always lead to better outcomes."

### Inferences
- loop-harness에 바로 적용할 수 있는 표준 패턴은 다음과 같습니다.
  1. 첫 실행 전용 초기화 단계(스펙/feature list JSON, init 스크립트, 첫 커밋)
  2. 반복 단계: "한 번에 한 항목 → 검증 → 커밋 → progress 갱신"
  3. 세션 시작 루틴: 상태 읽기, 그다음 smoke test
  4. 생성자와 분리된 evaluator, 그리고 사전 합의된 "완료 기준"(sprint contract)
- Managed Agents의 session(append-only 이벤트 로그), stateless harness, `execute(name,input)->string` 샌드박스 인터페이스는 Go 백엔드의 도메인 모델(Run/Event/Executor)로 그대로 옮기기 좋은 레퍼런스 아키텍처입니다. 특히 에이전트 CLI(Claude Code, Codex 등)를 교체 가능한 "harness 구현체"로 보는 meta-harness 관점이 tool-agnostic 목표와 일치합니다.
- 모델이 바뀌면 하네스 구성요소를 하나씩 끄고 성능을 재는 ablation 기능을 제품에 넣는 것도 Anthropic 원칙에 부합합니다.

### Gaps
- "Building a C compiler with a team of parallel Claudes"(2026-02-05), "Code execution with MCP"(2025-11-04), "Advanced tool use"(2025-11-24), "How we contain Claude across products"(2026-05-25), Claude Code auto mode(2026-03-25) 본문은 이번 조사 예산 안에서 읽지 못했습니다. 제목과 날짜만 engineering 인덱스로 확인했습니다.
- 2026-05 Managed Agents 후속 기능(dreaming/outcomes/multiagent)은 2차 출처 언급만 있어 1차 확인을 못 했습니다.

## Q3. Claude Code 하네스 구조 (CLAUDE.md/메모리, hooks, subagents, skills, plan mode, permissions, headless, 검증)

### Takeaway
현재(2026-11) Claude Code 공식 문서는 하네스를 "결정적(harness가 강제) vs 비결정적(모델이 판단)" 두 축으로 정리합니다.
- **비결정적**: CLAUDE.md, 프롬프트, skill은 모델이 따를지 판단함
- **결정적**: hook, deny 규칙, sandbox, managed settings, Stop hook, `--max-turns`/`--max-budget-usd`는 하네스가 강제함

가장 중요한 단일 조언은 "pass/fail이 나오는 체크를 줘서 루프를 스스로 닫게 하라"입니다. 과거 조언 일부(컨텍스트 공격적 관리, IMPORTANT 대문자 강조, "loop until done" 지시)는 공식적으로 폐기됐습니다.

### Cited Findings
- 에이전틱 루프는 gather context → take action → verify results → 반복이고, 사용자가 언제든 끼어들 수 있음. 공식 정의: "Claude Code is the layer around the model that provides the tools and manages the context the model sees. This surrounding layer is what the term agentic harness refers to." — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- 도구 카테고리는 file ops, search, execution, web, code intelligence이고, 여기에 subagent·질문·오케스트레이션 도구가 더해짐 — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- 시작 시 로드되는 것 — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
  - CLAUDE.md(저장소에 AGENTS.md가 있으면 그것도 읽을 수 있음)
  - auto memory `MEMORY.md`의 처음 200줄 또는 25KB
  - skill 설명(본문은 필요할 때 로드)
  - MCP 도구 정의는 기본적으로 지연 로드(tool search)
- 세션은 `~/.claude/projects/`에 JSONL로 저장되고 resume/fork/rewind가 가능함. 파일 편집 전 스냅샷(checkpoint)을 뜨지만, 원격 시스템(DB/배포)은 되돌릴 수 없음 — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- Compaction — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
  - 자동으로 일어남
  - CLAUDE.md의 "Compact Instructions" 섹션이나 `/compact <focus>`로 무엇을 보존할지 제어함
  - 반복적으로 thrashing하면 몇 번 시도 후 멈추고 오류를 냄
- 권한 모드 — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
  - Auto: 분류기가 위험한 행동을 차단함. v2.1.283 이상에서는 터미널/VS Code 기본 모드
  - Manual
  - Accept edits
  - Plan: 탐색과 계획만 하고 소스는 편집하지 않음
- 결정적/비결정적 표(2026 문서) — [Best practices](https://code.claude.com/docs/en/best-practices)
  - "When it's done": 비결정적은 증거 요청, `/goal`, `/code-review` / 결정적은 테스트, typecheck, `Stop` hook, CI
  - "How much it spends": 비결정적은 effort / 결정적은 `--max-turns`, `--max-budget-usd`
  - "Who has the last word": 비결정적은 Claude의 요약 / 결정적은 branch protection, 머지를 승인하는 사람
- "Give Claude a check to run": "Claude stops when the work looks done." 체크가 없으면 사람이 테스트 스위트 역할을 하게 됨. 테스트, 빌드 exit code, linter, fixture diff 스크립트, 스크린샷 등 pass/fail 체크를 주면 Claude가 스스로 루프를 닫음. CLAUDE.md에 넣을 가장 유용한 한 줄은 그 체크를 실행하는 명령 — [Best practices](https://code.claude.com/docs/en/best-practices)
- **Stop hook**으로 완료를 강제하는 방법 — [Best practices](https://code.claude.com/docs/en/best-practices)
  - 턴 종료 시 스크립트를 실행하고, exit 2면 stderr를 이유로 Claude를 다시 일하게 함
  - 도구 호출 없이 8번 연속 차단되면 override됨. 상한은 `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`로 조정
- 독립 리뷰 — [Best practices](https://code.claude.com/docs/en/best-practices)
  - `/code-review`는 대화를 보지 않은 fresh subagent가 diff를 리뷰함
  - `/verify`는 앱을 빌드·실행해 결과를 확인함
  - 스펙 대조 리뷰 프롬프트에는 "skip style"을 넣을 것(찾으라고 하면 뭐든 찾아내 과잉수정이 생김)
  - 쓰기 세션과 리뷰 세션을 분리하는 writer/reviewer 패턴
- Hooks — [Best practices](https://code.claude.com/docs/en/best-practices)
  - PreToolUse/PostToolUse/Stop/SessionStart 등 루프의 고정 지점에서 실행됨
  - 진짜로 강제하려면 `command`/`http` hook을 쓸 것. `prompt`/`agent` hook은 모델이 판단하므로 비결정적임
  - `onFailure: "block"`을 쓸 수 있음
  - 예외가 0인 규칙에만 쓸 것("forty of them"이면 대부분 선호사항임)
- 원칙: "If something has to happen every time, make it a hook, a deny rule, or a managed setting. CLAUDE.md is for guidelines" — [Best practices](https://code.claude.com/docs/en/best-practices)
- CLAUDE.md — [Best practices](https://code.claude.com/docs/en/best-practices)
  - 짧게 유지하고 코드에서 알 수 없는 것만 적을 것(명령, 컨벤션, gotcha)
  - memory 가이드는 200줄 미만을 권장함
  - 개인 설정은 `CLAUDE.local.md`에 둘 것
  - 아키텍처나 파일 레이아웃 설명은 불필요함(Claude가 직접 읽음)
- Skills — [Best practices](https://code.claude.com/docs/en/best-practices)
  - 팀 절차는 `.claude/skills/<name>/SKILL.md`에 둠
  - 이름과 설명만 상시 로드되므로 CLAUDE.md보다 가벼움
  - 부작용이 있는 워크플로(release 등)는 `disable-model-invocation: true`로 사용자만 호출하게 함
- Subagents — [Best practices](https://code.claude.com/docs/en/best-practices)
  - 백그라운드에서 실행됨
  - 병렬 작업, 시끄러운 출력 격리, 독립 리뷰에 유용함
  - 비용은 토큰이고, 컨텍스트를 다시 모아야 해서 더 느릴 수 있음
  - `.claude/agents/reviewer.md`에 `tools: Read, Grep, Glob`만 주면 하네스가 편집을 금지하는 리뷰어가 됨
  - 단순 작업은 `model: haiku`
- Auto mode는 분류기 판단이라 틀릴 수 있음. deny 규칙과 PreToolUse hook이 분류기보다 먼저 적용되므로 "절대 금지" 규칙은 deny나 hook에 둘 것 — [Best practices](https://code.claude.com/docs/en/best-practices)
- 무인 실행 전 체크리스트 5가지 — [Best practices](https://code.claude.com/docs/en/best-practices)
  1. done이 무엇인지 아는가(goal 조건, 스펙 파일, 통과시켜야 할 실패 테스트)
  2. 스스로 검증할 수 있는가(테스트, Stop hook, 리뷰 단계)
  3. 무엇을 만질 수 있는가(권한, sandbox, worktree, push할 수 있는 브랜치)
  4. 무엇이 멈추게 하는가(`--max-turns`, `--max-budget-usd`, 지출 한도)
  5. 결과는 어디로 가고 누가 승인하는가(PR, 그다음 사람이 머지 승인)
- Headless — [Best practices](https://code.claude.com/docs/en/best-practices)
  - `claude -p`를 CI, git hook, 셸 루프에서 씀
  - `--bare`는 hook/skill/plugin/MCP/CLAUDE.md를 모두 건너뛰어 어디서나 같은 동작을 보장함. 필요한 것은 `--append-system-prompt`, `--mcp-config`, `--settings`로 명시적으로 주입함
  - `/loop 15m ...`으로 주기 실행함
  - git worktree로 병렬 세션을 돌림
- 세션 조향 — [Best practices](https://code.claude.com/docs/en/best-practices)
  - 처음 1분을 지켜보고 벗어나면 Esc
  - 두 번 교정해도 안 되면 세 번째 교정 대신 `/clear` 후 더 나은 프롬프트
  - 스펙 먼저 브레인스토밍해 `SPEC.md`를 만들고, 깨끗한 세션에서 구현
- **"What you can stop doing"**(공식적으로 폐기된 조언) — [Best practices](https://code.claude.com/docs/en/best-practices)
  - 컨텍스트를 공격적으로 관리하기: 대부분 1M 창과 자동 compaction이 있음
  - IMPORTANT나 대문자 강조: 과잉 적용을 유발함. hook이나 deny 규칙으로 옮길 것
  - 컨텍스트 보호용 subagent
  - "subagent에 맡기면 답이 더 좋다": 그렇지 않음. fresh 리뷰어만 효과가 있음
  - "think harder" 문구: effort 설정이 대신함
  - "loop until done, don't be lazy": 과잉작업을 유발함. 대신 체크, `/goal`, Stop hook을 쓸 것
  - CLAUDE.md에 아키텍처 설명

### Inferences
- loop-harness가 에이전트 비종속으로 "체크 기반 종료"를 구현하려면 Claude Code의 Stop hook(exit 2 → 피드백 재주입, 연속 차단 상한)을 하네스 레벨 일반 개념으로 추상화하면 됩니다. 구성 요소는 "verify 명령 → 실패 시 출력을 다음 iteration 입력으로 → 최대 재시도 상한"입니다.
- 결정적/비결정적 표(무엇을/무엇을 만질지/언제 끝/얼마나 쓸지/최종 승인)는 하네스 설정 스키마를 설계할 때 그대로 차원(axis)으로 쓸 만합니다.
- AGENTS.md 호환을 공식 지원하므로, 도구 비종속 하네스는 AGENTS.md를 1차 컨텍스트 파일로 두고 CLAUDE.md를 호환 대상으로 다루는 편이 표준적입니다.

### Gaps
- 공식 문서는 날짜가 없는 living doc입니다. 2026-11-18 조회 시점의 내용이며 버전 표기(v2.1.283)만 확인됩니다.
- hooks 이벤트 전체 목록과 subagent frontmatter 전체 필드는 이번에 개별 문서(hooks, sub-agents)를 읽지 않아 상세가 비어 있습니다.

## Q4. 평가(evals), 검증, 피드백 루프, 컨텍스트 관리(compaction, progress 파일, git as state)

### Takeaway
- **컨텍스트**: 유한한 attention budget이므로 "가장 작은 고신호 토큰 집합"을 just-in-time으로 로드함. 장기 작업은 compaction, 구조화된 노트, subagent 세 가지로 대응함.
- **메모리**(2026-11): 4계층(transcript, notes, 구조화 표현, 주입 컨텍스트)으로 나누고, 상단 주입층은 작게, 오프라인 통합("dreaming")으로 관리함.
- **Evals**: task/trial/grader/transcript/outcome 용어로 정의함. 실패 사례 20~50개로 시작하고, 경로가 아니라 결과를 채점하며, pass@k와 pass^k를 구분하고, "Read the transcripts!"를 강조함.

### Cited Findings
**Context engineering (2025-09-29)**
- 정의: context engineering은 추론 시 들어가는 모든 것(시스템 프롬프트, 도구, 외부 데이터, 메시지 히스토리)을 매 턴 큐레이션하는 일 — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- context rot: 토큰이 늘수록 회상 정확도가 떨어짐(n² attention, 장문 학습 데이터 부족). 절벽이 아니라 점진적 저하임 — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- 시스템 프롬프트는 "right altitude"로 씀. 하드코딩된 if-else와 모호한 지침 사이의 중간이고, 최소 프롬프트로 시작해 관찰된 실패에 따라 추가함 — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- 도구가 겹치는 비대한 도구 세트를 피할 것. 사람 엔지니어가 어떤 도구를 쓸지 고를 수 없으면 에이전트도 못 고름. 예시는 정형적이고 다양한 소수로 둠 — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Just-in-time 로드 — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
  - 경로나 쿼리 같은 가벼운 참조만 들고 있다가 도구로 필요할 때 로드함
  - progressive disclosure
  - 하이브리드: CLAUDE.md는 미리 로드하고 나머지는 탐색함
- 장기 작업 기법 — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
  - **Compaction**: 아키텍처 결정과 미해결 버그는 보존하고 중복 도구 출력은 버림. recall 먼저 튜닝하고 precision은 그다음. 오래된 도구 결과 지우기가 가장 안전한 형태
  - **Structured note-taking**: NOTES.md, to-do. memory tool
  - **Sub-agent**: 수만 토큰을 쓰고 1,000~2,000 토큰 요약을 반환함
- 각 기법의 용도 — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
  - compaction: 긴 대화형 작업
  - 노트: 마일스톤이 있는 반복 작업
  - 멀티에이전트: 병렬 리서치
- 핵심 인용: "find the smallest set of high-signal tokens that maximize the likelihood of your desired outcome" — [Context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- (주의) 2026 Claude Code 문서는 "컨텍스트 창이 주 제약이니 공격적으로 관리하라"를 폐기된 조언으로 분류함. 1M 창과 자동 compaction 때문임. 다만 `/clear`로 컨텍스트를 집중시키는 것은 여전히 권장함 — [Best practices](https://code.claude.com/docs/en/best-practices)

**Git과 파일을 상태로 쓰기**
- progress 파일과 git log 조합으로 새 세션이 상태를 파악하고, git commit으로 나쁜 변경을 되돌림 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- sprint contract와 에이전트 간 통신을 파일로 함 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- 세션 이벤트 로그를 컨텍스트 밖 영속 상태로 둠. compaction은 비가역적이기 때문 — [Managed Agents](https://www.anthropic.com/engineering/managed-agents)

**The components of agent memory (2026-11-06, Lance Martin) — 최신**
- 4계층: transcripts(검색 가능한 이벤트 로그), memory notes, memory representations(위키 등 링크된 페이지), injected context(매 세션 시작 시 로드되는 작은 층). 계층 간 이동은 오프라인 통합 "dreaming"이 담당함 — [Agent memory](https://www.anthropic.com/engineering/components-of-agent-memory)
- 권고 — [Agent memory](https://www.anthropic.com/engineering/components-of-agent-memory)
  - 강한 모델은 고정 규칙보다 판단에 맡길 때 더 좋음. "record only failures" 같은 규칙은 Opus 5와 Opus 4.7의 이득을 줄였음
  - 무분별하게 로깅하지 말 것: Sonnet 3.7은 413개 파일을 만들었고 stale해짐
  - "tell Claude what memory is for, but not how to use it"
  - 엔티티당 한 페이지: 계약 태스크에서 보존된 사실이 41/57에서 57/57로 늘었음
- 주입 컨텍스트는 작게 유지할 것. 잘못된 노트가 모든 세션에 전파되기 때문. 병렬 에이전트는 쓰기를 분할해야 함. 메모리 지시를 바꿀 때마다 전후를 측정할 것 — [Agent memory](https://www.anthropic.com/engineering/components-of-agent-memory)
- 더 긴 지시 파일은 준수율을 낮춤 — [Agent memory](https://www.anthropic.com/engineering/components-of-agent-memory)

**Demystifying evals for AI agents (2026-01-09)**
- 용어 — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
  - task, trial, grader, transcript(trace), outcome(환경의 최종 상태, 에이전트의 주장이 아님)
  - evaluation harness
  - **agent harness(scaffold)**: 에이전트를 평가한다는 것은 하네스와 모델을 함께 평가하는 것
  - eval suite
- Grader 3종: code-based(빠르고 객관적이지만 brittle), model-based(유연하지만 보정이 필요함), human(골드 스탠더드지만 느림). 점수는 weighted, binary, hybrid로 결합 — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- capability eval은 낮은 통과율에서 시작하고, regression eval은 약 100%를 유지함. capability eval이 포화되면 regression으로 "졸업"시킴 — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- pass@k(k번 중 1번 이상 성공)와 pass^k(k번 모두 성공). 회당 75% 성공이면 3연속 성공 확률은 약 42% — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- 코딩 에이전트 eval — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
  - 결정적 테스트로 정확성을 보고, LLM 루브릭으로 품질을 봄
  - 필요하면 정적분석(ruff, mypy, bandit), 상태 체크, 도구 호출 요건을 추가함
- 로드맵 — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
  - 실제 실패 20~50개로 일찍 시작할 것
  - 수동 체크와 버그 리포트를 태스크로 바꿀 것
  - 두 전문가가 같은 판정을 내릴 만큼 명확한 태스크와 참조 해답을 둘 것. 0% 통과는 태스크나 grader 버그를 의심할 것
  - 행동이 일어나야 할 때와 일어나면 안 될 때를 모두 넣어 균형을 맞출 것
  - trial마다 깨끗한 환경으로 격리할 것
  - 경로가 아니라 결과를 채점하고, 부분 점수를 주고, LLM judge를 사람 판정으로 보정하고 "Unknown" 옵션을 줄 것
  - 치팅에 강하게 만들 것: CORE-Bench는 grader 버그로 42%였다가 고친 뒤 95%가 됨
  - transcript를 읽을 것
  - 포화를 감시할 것
  - eval-driven development
- 프레임워크 예시: Harbor, Braintrust, LangSmith, Langfuse, Arize. 프레임워크는 빨리 고르고 태스크 품질에 투자할 것 — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- "Swiss cheese" 모델: automated eval, 프로덕션 모니터링, A/B, 사용자 피드백, transcript 리뷰를 겹겹이 쌓음 — [Demystifying evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

**검증 / 피드백 루프 요약**
- 검증 수단 3종: 규칙 기반(lint/test), 시각(스크린샷), LLM judge(제한적으로) — [Agent SDK blog](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)
- 작업자와 평가자를 분리하는 것이 자기평가 편향에 대한 강한 레버임. 다만 evaluator는 튜닝이 필요함 — [Harness design](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- 완료 표시 전에 E2E 자기검증을 할 것 — [Effective harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

### Inferences
- loop-harness의 "verify" 단계는 다층으로 설계하는 것이 Anthropic 표준과 맞습니다.
  1. 결정적 체크(test/lint/typecheck exit code): 필수
  2. E2E/시각 체크: 선택
  3. 독립 evaluator 에이전트(fresh context, 읽기 전용): 선택, 비용 대비 효과가 있을 때
- 루프 실행 기록을 trial/transcript/outcome 스키마로 저장하면, 하네스 자체를 eval 하네스로도 쓸 수 있습니다(pass^k 측정, 하네스 구성 ablation).
- 상태 저장은 "append-only 이벤트 로그 + git + progress/feature-list 파일" 조합이 Anthropic이 반복해 제시한 표준입니다.

### Gaps
- "Quantifying infrastructure noise in agentic coding evals"(2026-02-05)와 "Building strong safety evaluations"(2026-11-13) 본문은 읽지 못했습니다.

## Q5. Anthropic이 오픈 표준에 기여한 것 (MCP, Agent Skills, Agentic AI Foundation)

### Takeaway
- **MCP**: Anthropic이 만들었고 2025-12-09 Linux Foundation 산하 Agentic AI Foundation(AAIF)에 기증함. AAIF는 Anthropic, Block, OpenAI가 공동 설립했고, 창립 프로젝트는 MCP, goose, AGENTS.md.
- **Agent Skills(SKILL.md)**: 2025-12-18 agentskills.io에서 오픈 표준으로 공개됨. Codex, Copilot, Cursor, Gemini CLI 등이 채택함.
- **AGENTS.md**: OpenAI가 기여함. Claude Code도 읽을 수 있음.

결과적으로 MCP(도구/통합), SKILL.md(절차 지식), AGENTS.md(프로젝트 지시)가 에이전트 비종속 하네스의 사실상 표준 3종입니다.

### Cited Findings
- 2025-12-09 Anthropic 공지: MCP를 AAIF(Linux Foundation의 directed fund)에 기증함. AAIF는 Anthropic, Block, OpenAI가 공동 설립함 — [Anthropic: Donating MCP & establishing AAIF](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation)
- Linux Foundation 발표: 창립 기여 프로젝트는 Anthropic의 MCP, Block의 goose, OpenAI의 AGENTS.md — [Linux Foundation press release](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- (날짜 충돌) 한 2차 출처는 MCP 기증을 2026-05-19로 적었으나, Anthropic과 LF 1차 자료 모두 2025-12이므로 오류로 판단함 — [Business Tech Navigator](https://businesstechnavigator.com/news/agentic-ai-foundation-mcp-linux-foundation-open-standard)
- Agent Skills — [Agent Skills (engineering)](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
  - 2025-10-16에 처음 공개됨
  - 2025-12-18 업데이트로 "cross-platform portability를 위해 오픈 표준으로 공개(agentskills.io)"
  - 형식: `SKILL.md` + YAML frontmatter(필수 `name`, `description`)
  - 3단계 progressive disclosure: 메타데이터, 본문, 링크된 파일
  - 결정적 작업을 위해 스크립트를 번들할 수 있음
  - 보안: 신뢰할 수 있는 출처에서만 설치하고, 외부 출처는 감사할 것
- agentskills.io — [agentskills.io](https://agentskills.io)
  - "originally developed by Anthropic, released as an open standard"
  - 디렉터리 구조: `SKILL.md`, `scripts/`, `references/`, `assets/`
  - 단계: Discovery, Activation, Execution
  - GitHub `agentskills/agentskills`에서 기여를 받음
- OpenAI Codex 채택: Simon Willison은 2025-12-19 시점 "OpenAI는 아직 공식 발표 전"이라고 썼고, 이후 Codex 문서에 Skills가 추가되고 agentskills.io에 Codex 로고가 올라갔다고 업데이트함 — [Simon Willison, 2025-12-19](https://simonwillison.net/2025/Dec/19/agent-skills/)
- 2026-06 기준 클라이언트 쇼케이스에 약 40개 채택사(GitHub Copilot, VS Code, Cursor, Codex, Gemini CLI, Goose, OpenCode 등)가 있다는 주장이 있음. 2차 출처라 수치는 근사로 취급 — [Agent Skills open standard (blog)](https://codex.danielvaughan.com/2026/05/05/agent-skills-open-standard-portable-skills-codex-cli-cross-agent/)
- Claude Code는 저장소의 AGENTS.md를 CLAUDE.md 대신 읽을 수 있음 — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
- MCP의 역할은 Slack/GitHub/Drive 등 표준 통합이며, 인증과 API 호출을 처리함 — [Agent SDK blog](https://claude.com/blog/building-agents-with-the-claude-agent-sdk)

### Inferences
- tool-agnostic 하네스라면 다음 순서로 지원하는 것이 업계 표준과 정렬됩니다.
  1. 프로젝트 지시는 AGENTS.md, 에이전트별 파일(CLAUDE.md 등)은 심볼릭/호환
  2. 절차 지식은 SKILL.md 디렉터리
  3. 도구와 통합은 MCP
- 세 표준 모두 특정 벤더에 종속되지 않는 거버넌스(AAIF/LF 또는 오픈 스펙 저장소)를 갖고 있어 장기적으로 안전한 선택입니다.

### Gaps
- Agent Skills 스펙이 AAIF로 이관됐는지 여부를 1차 자료로 확인하지 못했습니다. agentskills.io에는 "Anthropic이 개발, 오픈 표준, 기여 환영"이라고만 적혀 있습니다.
- agentskills.io 스펙의 선택 필드(license, compatibility, metadata, allowed-tools 등) 전체 목록은 specification 페이지를 따로 읽지 않아 미확인입니다.
- 2026년 AAIF에 추가로 편입된 프로젝트는 확인하지 못했습니다.
