# 한국 IT 기업의 AI 코딩 에이전트 루프/하네스 실무 (2026-11-18 기준)

> 표기: [공식] = 회사 공식 기술블로그/저장소/발표, [개인] = 개인 블로그·커뮤니티, [기사] = 언론/애그리게이터. 날짜는 원문 게시일.
> 조사 범위: 약 17회 검색/페치. 일부 회사(쿠팡, 당근, 뱅크샐러드, 쏘카, 삼성SDS, LG CNS, NHN, 카카오뱅크)는 관련 공식 글을 찾지 못함 → Gaps 참고.

## Q1. 주요 한국 기업이 공개한 Claude Code / Codex / Cursor 도입, AGENTS.md·CLAUDE.md 컨벤션, 루프, 하네스, MCP, 검증 게이트

### Takeaway
가장 구체적인 공개 자료는 카카오페이(하네스 구축→축소 회고, SDD/spec-kit, MCP Toolkit), LY/LINE(Claude Code Action 기반 중앙집중 AI 코드리뷰 플랫폼), 토스(앱인토스 공식 하네스 저장소 + 아키텍처 비유 글), 우아한형제들(Cursor rules/skills + 전처리 스크립트로 컨텍스트 96% 절감), 컬리(Plan→Act→Verify와 디렉토리별 CLAUDE.md), 네이버 D2(Claude Code 기반 멀티 에이전트 프레임워크 NaverMadCat)이다. 공통 축은 "프롬프트가 아니라 구조(hooks·스크립트·실행결과 기반 검증)로 강제한다"는 점이다.

### Cited Findings

**카카오페이 (tech.kakaopay.com) [공식]**
- 「AI Agent, 넌 누구냐 1 - Tool, MCP, Skill, Harness는 무엇인가」(마쉬, FE, 2026-09-01): Harness를 "모델을 똑똑하게 만드는 대신 모델이 일하는 환경을 설계"하는 것으로 정의, 역할은 제어(Control)·감시(Monitoring)·개선(Feedback), 핵심 원칙은 "실수를 프롬프트가 아니라 구조로 막는다". 루프 종료 기준 3가지(모델 자체 완료 선언 / 최대 반복·토큰 예산 / 테스트 등 외부 검증) 중 외부 검증을 가장 신뢰할 만하다고 정리. Skill = SKILL.md 기반 "도구 활용 매뉴얼", 점진적 공개(progressive disclosure). — [카카오페이 기술블로그](https://tech.kakaopay.com/post/ai-agent-1/)
- 「AI Agent, 넌 누구냐 2 - 네이티브 개발자를 위한 FE 하네스를 만들고, 레거시가 되기까지」(홀트 hoult.jun, 오프라인결제 클라이언트팀, 2026-11-03) — 이번 조사에서 가장 실무적인 자료: — [카카오페이 기술블로그](https://tech.kakaopay.com/post/ai-agent-2/)
  - 목적: 네이티브(iOS/Android) 개발자도 FE 팀 기준으로 FE 작업을 하도록 암묵지(워크플로·컨벤션·디자인시스템·이벤트로깅 규칙)를 하네스화. 긴 단일 룰 파일은 에이전트가 놓치므로 역할·경로별로 쪼개고 서브에이전트·스킬로 필요한 것만 로드. 팀 플러그인으로 배포(부분 설치 가능).
  - Init 스킬 + 프리셋: 프로젝트의 디자인시스템·코드 패턴 탐지 → 경로 스코프 룰 파일(React, SCSS, API, 이벤트로깅) 생성, hard rule / soft 권고 분리, 프리셋 메타데이터 기록 및 검증.
  - 디자인→코드 스킬 4단계: 읽기전용 분석 에이전트(코드 작성 금지, 컴포넌트/토큰 매핑) → 구현 → 메인 세션이 직접 타입체크·lint → 서로 다른 관점(유지보수성/컨벤션/정확성) 리뷰어 3명 병렬 리뷰. 별도 오케스트레이터 없이 메인 세션이 직접 위임(핸드오프·컨텍스트 손실 방지).
  - Hooks 7개 스크립트 × 4 이벤트: `UserPromptSubmit`(키워드→스킬 라우팅), `PreToolUse`(잘못된 서브에이전트 위임 차단, 선행 산출물 없는 단계 차단, 프로젝트 룰 주입), `PostToolUse`(lint/타입체크 결과 기록, 실패 시 수정 단계로 되돌림), `Stop`(필수 단계 남았으면 종료 차단).
  - 검증: lint/typecheck/test 결과를 에이전트 주장(self-report)이 아니라 실제 명령 실행 결과로 기록.
  - 플러그인 거버넌스: 플러그인별 독립 버전, Slack/PR로 변경 공지, 로컬 테스트 후 배포, 분할 터미널(한 세션은 플러그인 개발, 다른 세션은 사용).
  - 실패 사례: (1) 위임 가드 hook이 잘못된 출력 포맷으로 실 세션에서 무동작(유닛테스트는 통과) (2) 에이전트가 `| tail`을 붙여 exit code가 tail 것이 되어 lint 실패가 통과로 기록 (3) lint 에러 선존재 확인용 `git stash`가 공유 워킹트리의 다른 세션 미커밋 작업을 날림(지시문으로만 금지, 스크립트 강제는 아직) (4) 에이전트가 정상 케이스만 테스트하고 "전부 검증됨" 보고 → 새 세션이 남은 문제 발견.
  - "레거시화": 2026년 2월→10월 모델 향상으로 에이전트 21개·스킬 29개 → 11개·17개로 축소. 여전히 유효: 코드에서 추론 불가능한 팀 컨벤션, 검증·릴리즈 게이트 같은 강제 장치. 낡은 것: 단계 쪼개기, 강압적 프롬프트 문구, 페르소나 기반 라우팅(노이즈·토큰 비용).
  - 미해결 과제: 에이전트의 검증 과장, 세션 간 교훈 기록 방법(컨텍스트 비용/낡은 지시 문제), 로컬 산출물 누적, 하네스 변경을 판단할 정량 지표 부재("지표 없으면 하네스는 커지기만 한다").
- 「SDD(spec-kit) 에이전트 코딩 실전기」(wayne.cai, AI플랫폼팀, 2026-03-13): constitution → specify↔clarify(최대 5개 질문) → plan → tasks → analyze(constitution 대비 문서 일관성 점검) → implement. 산출물 문제 시 코드를 직접 고치지 않고 spec으로 돌아가 하위 단계 재생성. 실패모드: tasks 집중으로 constitution 무시, 문서-코드 불일치, spec 항목이 tasks에서 누락(예: MCP 호출), 작은 변경에 파이프라인을 탈지 기준 부재, 토큰 비용. — [카카오페이 기술블로그](https://tech.kakaopay.com/post/ifkakao-agentic-coding/)
- 「AI 에이전트와 카카오페이 결제 오픈 API 연동하기: MCP Agent Toolkit 개발기」(2025-08-06): `@kakaopay-develop/agent-toolkit`, `@kakaopay-develop/mcp` 패키지로 멀티 프레임워크 지원. — [카카오페이 기술블로그](https://tech.kakaopay.com/post/kakaopay-mcp-agent-toolkit/)

**카카오 [공식/기사]**
- 카카오는 2025-08-13 MCP 기반 개방형 플랫폼 PlayMCP를 베타 공개(remote MCP만 등록 가능). — [다음 뉴스](https://v.daum.net/v/8IAEjSJqQS) [기사]
- tech.kakao.com 본사 블로그에서 코딩 에이전트 하네스 관련 글은 이번 검색에서 확인되지 않음.

**토스 [공식]**
- 「소프트웨어 3.0 시대를 맞이하며」(김용성, 토스페이먼츠 Node.js, 2026-01-26): "Claude Code는 본질적으로 Claude 모델을 위한 Harness". 매핑: Slash Command=Controller, Sub-agent=Service Layer, Skill=도메인 컴포넌트(SRP), MCP=Adapter, CLAUDE.md=package.json(안정적 규칙만; 현재 티켓 같은 동적 정보는 대화/서브에이전트로). 실천: 결정적 로직은 스크립트로(예: 브랜치 네이밍 스크립트), SKILL.md 진입점 + `references/` on-demand(Facade/progressive disclosure), HITL은 비가역·고위험·모호한 결정에만, 워크플로가 읽을 파일을 미리 물어 토큰 비용 추정. 안티패턴: God Skill, Spaghetti CLAUDE.md, MCP 대신 하드코딩된 도구 호출, 순환 Skill 의존, 중복 프롬프트. "Exception이 Question으로 바뀐다." — [toss.tech](https://toss.tech/article/44539)
- `toss/apps-in-toss-harness` (BSD-3, pnpm 모노레포, 조회 시점 커밋 462·스타 194): "빈 디렉토리부터 미니앱 출시까지 에이전트를 떠나지 않고" 완주하는 하네스. Claude Code 플러그인 마켓플레이스(`ait@apps-in-toss`) 배포, Codex·Cursor도 지원(명령명이 달라 자연어 요청 권장). 슬래시 커맨드 파이프라인 `/ait:plan`→`/ait:new`→`/ait:design`·`/ait:feature`·`/ait:server`→`/ait:test-on-device`·`/ait:debug`→`/ait:launch`. MCP: `ait-docs`(문서, 로그인 불요), `ait-console`(OAuth, 앱 등록/번들 업로드), `llms.txt` 폴백. 디버그 MCP 서버 + 폰 릴레이 + 테스트러너 패키지. 가드레일: 심사 제출은 `/ait:launch`에서 전송 내용을 보여주고 사용자 승인 후에만, 출시/롤백/프로모션은 사람이 콘솔에서, Deploy Key 없이 OAuth만. 루트에 CLAUDE.md(AGENTS.md는 확인 안 됨). — [GitHub](https://github.com/toss/apps-in-toss-harness)

**우아한형제들 (techblog.woowahan.com) [공식]**
- 「하네스 엔지니어링으로 팀 맞춤형 AI 환경 구축하기」(이재홍, 파트너셀프서비스팀 FE, 2026-04-17): 도구는 Cursor. `.cursor/rules`(globs로 경로별 활성화, 프로젝트 특화 규칙만—LLM이 아는 건 넣지 않음), `.cursor/skills`(Swagger→TS 타입/Mock, React Query 훅 생성, 테스트 생성, MR 본문 작성, 컨벤션 리팩토링). 핵심 기법: 스킬 실행 전 전처리 스크립트가 코드베이스에서 메타데이터(클래스명·메서드명·엔드포인트)만 JSON으로 추출 → 전달 데이터 평균 약 96.5% 감소, 도구 호출 4회→1회(예: 41,944B→1,763B). 단, 토큰은 1토큰≈4B 추정치, 품질 지표는 체감 수준. — [우아한형제들 기술블로그](https://techblog.woowahan.com/26177/)
- 같은 블로그에 "AI가 규칙을 자꾸 어기던 문제 → 컨텍스트 엔지니어링·하네스 설계, 최신 모델엔 절차보다 성공 기준·검증 규칙을 명확히"라는 취지의 별도 글이 있다고 애그리게이터가 소개(원문 URL 미확인). — [Velopers](https://velopers.kr/blog/1)

**컬리 (helloworld.kurly.com) [공식]**
- 「Claude Code를 활용한 예측 가능한 바이브 코딩 전략」(박재영, 2025-12-17, Claude Code 2.0.67/Opus 4.5): 루프 Plan→Act→Verify, Plan 모드로 파일 변경 전 방향 확인. 루트 CLAUDE.md는 온보딩(명령어·구조·데이터 흐름)만, 영역 규칙(예: 결제 모듈 정수 통화 처리)은 디렉토리별 CLAUDE.md. Compact 전 결정사항을 `docs/decisions.md`에 기록(외부 작업 메모리), Todo로 다파일 변경 체크리스트화, Explore 서브에이전트 병렬 분석, PR 전 다중 페르소나 리뷰(보안/성능/유지보수/테스트), 룰 체커 스킬. 실패모드: Lost in the Middle, 학습데이터 편향(15턴에 걸쳐 Zustand 지시가 Redux 패턴으로 회귀), 오류 누적, 지시 희석, 다중 제약 누락. 한 줄 요약 "작게 나누고, 자주 확인하고, 반복은 스킬로". — [컬리 기술블로그](https://helloworld.kurly.com/blog/vibe-coding-with-claude-code/)

**LY Corporation / LINE [공식]**
- 「Claude Code Action: 조직 전반의 코드 품질을 지키는 AI 코드 리뷰 플랫폼화」(이동원, LINE NEXT DevOps; 게시일 미표기, 2025-12 중순~2026-01 초 데이터): Caller–Executor 구조 — 서비스 레포엔 얇은 caller 워크플로(`service_name`, `analysis_type`, `runner_type`)만, 중앙 DevOps 레포가 프롬프트·정책·권한·실행로직 소유(`configs/common/` + `configs/{service}/` 병합). 보안: PAT 대신 GitHub App의 PR 스코프 단기 토큰, 모델은 사내 프록시 경유 Amazon Bedrock. 프롬프트: 공통(SRP/DRY/네이밍/에러처리/보안, Blocking·Recommended·Suggestion·Tech Debt 4단계) + 서비스별(Kotlin/Spring null safety, JPA N+1 등). 사용자 `@claude` 코멘트가 출력 포맷을 덮어쓰지 못하도록 우선순위 역전 + "MANDATORY" 래퍼. "security" 키워드로 보안 리뷰(Critical~Low). fork PR은 `refs/pull/<N>/head` fetch로 해결. 약 1개월간 32개 레포·344회 리뷰, 사내 DORA 대시보드 "Build"에 편입. 정확도·오탐률·비용 지표는 미제시. — [LY 테크블로그(ko)](https://techblog.lycorp.co.jp/ko/building-ai-code-review-platform-with-claude-code-action)
- LINE 앱(iOS 약 235만 행, Android 약 268만 행) 대상 Claude Code 활용 기반: CLAUDE.md를 약 150행으로 줄이는 설계 원칙과 서브에이전트 구조(일본어 발표, 2026-02-18 추정 — 슬라이드 ID 기준). — [Speaker Deck](https://speakerdeck.com/lycorptech_jp/20260218e)
- LY는 최근 1년간 AI 작성 코드 비중 약 20%, 사내 문서를 AI가 이해하기 쉬운 형태로 구조화한 것이 기반이라고 설명. — [플래텀](https://platum.kr/archives/290451) [기사]

**네이버 [공식, 애그리게이터 경유 확인]**
- D2 「AI 에이전트 회사 차리기: 설립부터 어디서든 동기화까지」(2026-06-30, ENGINEERING DAY 2026 발표): Claude Code가 세션마다 초기화되는 문제 해결용 사내 멀티 에이전트 프레임워크 NaverMadCat — "비서실장" 에이전트가 10개 부서 에이전트 조율, 기기/환경 간 동일 설정·목표 동기화, 라이프사이클 hooks(시작/종료/에러), 에이전트 채용·해고·병합 같은 HR식 운영 커맨드, 슬래시 커맨드 + 대시보드. 원문: https://d2.naver.com/helloworld/3897377 — [pile 요약](https://piletech.kr/articles/111697)

**SK [공식]**
- SK AX 인사이트 「루프 엔지니어링: 하네스를 넘어 AI가 스스로 반복하며 일하는 구조」(2026-08-28) — Q4 참고. 자사 AXgenticWire Core(비개발자가 에이전트 구축·실행·평가·개선) 소개, 구체 고객 사례는 없음. — [SK AX](https://www.skax.co.kr/insight/trend/3840)
- SK DevOcean에 「진짜 AI 에이전트형 코딩 툴 'Claude-Code'」 글 존재(내용·날짜 미확인). — [DevOcean](https://devocean.sk.com/blog/techBoardDetail.do?ID=167718&boardType=techBlog)

### Inferences
- 한국 공개 사례는 대부분 **FE/클라이언트 팀 주도**(카카오페이, 우아한형제들, 컬리)이고, 백엔드·레거시 대상 하네스 공개 사례는 드물다 → loop-harness가 레거시 백엔드 사례를 공개하면 희소가치가 있음.
- 카카오페이 2편은 loop-harness 설계에 바로 쓸 수 있는 체크리스트: (a) 검증 결과는 실행 exit code로만 기록(파이프 `| tail` 차단, `set -o pipefail` 류), (b) 공유 워킹트리 파괴 명령(`git stash`, `reset`) 스크립트 차단 → worktree 격리, (c) Stop 게이트, (d) 하네스 컴포넌트에 "어떤 모델 한계를 가정하는지" 메타데이터를 달아 모델 업그레이드 때 제거 후보 판단, (e) 하네스 변경 효과 측정 지표.
- LY 사례는 "개인 도구 → 조직 표준 컴포넌트" 전환의 레퍼런스(중앙 정책 레포 + 얇은 caller + 단기 토큰 + 출력 포맷 강제).
- 도구 중립성: 토스 하네스가 Claude Code/Codex/Cursor 동시 지원 + MCP/`llms.txt` 폴백을 택한 것은 loop-harness의 tool-agnostic 방향과 일치.

### Gaps
- 쿠팡, 당근, 뱅크샐러드, 쏘카, NHN, 카카오뱅크, 삼성SDS, LG CNS의 공식 코딩 에이전트/하네스 글은 이번 검색에서 찾지 못함(존재하지 않는다는 뜻은 아님; 블로그 내부 검색 필요).
- 네이버 D2 원문은 직접 열람 못 함(애그리게이터 요약만). 네이버 클라우드의 사내 코딩 에이전트 정책 미확인.
- LY 한국어 글 게시일 미표기.

## Q2. 한국 컨퍼런스 발표 (if(kakao), DEVIEW, SLASH, 우아콘, AWS Summit Seoul 등)

### Takeaway
확인된 것은 if(kakao)25 카카오페이 AI 플랫폼 세션(Agentic Coding 포함)과 네이버 ENGINEERING DAY 2026의 NaverMadCat 발표 정도이며, SLASH/우아콘/DEVIEW 2026의 에이전틱 코딩 세션 목록은 확인하지 못했다.

### Cited Findings
- if(kakao)25 세션 「AI 플랫폼 하드웨어부터 코드까지: GPU, LLMOps, Agentic Coding으로 완성하는 AI 플랫폼」(카카오페이) — SDD/spec-kit 글의 참고자료로 링크됨. — [if(kakao) 세션](https://if.kakao.com/2025/session?sessionId=42); [카카오페이 기술블로그](https://tech.kakaopay.com/post/ifkakao-agentic-coding/)
- 네이버 ENGINEERING DAY 2026: NaverMadCat 멀티 에이전트 프레임워크 발표(D2 글 2026-06-30). — [pile](https://piletech.kr/articles/111697)
- LY Tech(일본) 발표: LINE 앱 대규모 레포용 Claude Code 활용 기반. — [Speaker Deck](https://speakerdeck.com/lycorptech_jp/20260218e)
- AWS Summit Seoul 2026에서 "Nova Act & Strands Agent 실전: AI 에이전트로 개발 워크플로 자동화하기" 세션이 있었다는 발표자 본인 언급(개인 프로필, 소속 기업 불명). — [LinkedIn](https://kr.linkedin.com/in/jina-bae) [개인]
- 커뮤니티 밋업: 인프런 오프라인 「하네스, 에이전트에게 건네는 계약서」(오픈소스 Agent OS 'Ouroboros' 개발자 이재규). — [인프런](https://www.inflearn.com/course/이재규ouroboros-빌더밋-하네스) [개인/커뮤니티]
- "한국 최초 랄프톤(Ralph-thon)" 후기 — 하네스 엔지니어링 시대를 주제로 한 해커톤 후기. — [Substack](https://briandwjang.substack.com/p/8d3) [개인]

### Inferences
- 한국 대형 컨퍼런스에서 "하네스/루프"를 제목에 단 기업 세션은 아직 드물고, 커뮤니티 밋업·해커톤(랄프톤 = Ralph loop 기반)이 먼저 움직이고 있다.

### Gaps
- SLASH 26(토스), 우아콘 2026, DEVIEW, if(kakao) 2026의 세션 목록·영상 미확인 → 각 공식 사이트 직접 확인 필요.

## Q3. 공통 패턴, 교훈, 실패 모드, 거버넌스/보안

### Takeaway
공통 패턴은 (1) 짧고 안정적인 루트 지침 + 경로/디렉토리 스코프 규칙, (2) 반복 작업의 스킬화와 결정적 로직의 스크립트화, (3) 읽기전용 분석/구현/검증/리뷰의 역할 분리, (4) 자기보고가 아닌 실행 결과 기반 검증 게이트, (5) 비가역 작업은 사람 승인. 거버넌스는 중앙 정책 레포·플러그인 버전관리·단기 토큰·사내 프록시로 수렴한다.

### Cited Findings
- 짧은 루트 지침 + 스코프 규칙: 컬리(루트 CLAUDE.md는 온보딩만, 디렉토리별 CLAUDE.md) — [컬리](https://helloworld.kurly.com/blog/vibe-coding-with-claude-code/); LINE(CLAUDE.md 약 150행) — [Speaker Deck](https://speakerdeck.com/lycorptech_jp/20260218e); 우아한형제들(globs 경로 활성화) — [우아한형제들](https://techblog.woowahan.com/26177/); 카카오페이(경로 스코프 룰 파일, hard/soft 분리) — [카카오페이](https://tech.kakaopay.com/post/ai-agent-2/)
- 결정적 로직은 스크립트로: 토스(브랜치 네이밍 스크립트) — [toss.tech](https://toss.tech/article/44539); 우아한형제들(메타데이터 추출 전처리) — [우아한형제들](https://techblog.woowahan.com/26177/)
- 실행결과 기반 검증 & 자기검증 불신: 카카오페이(실행 결과 기록, Stop hook, 새 세션 재검증) — [카카오페이](https://tech.kakaopay.com/post/ai-agent-2/); 루프 종료 기준 중 외부 검증이 최우선 — [카카오페이](https://tech.kakaopay.com/post/ai-agent-1/); builder와 verifier 분리 — [SK AX](https://www.skax.co.kr/insight/trend/3840)
- 스펙 중심 루프: spec 수정 → 하위 산출물 재생성, analyze 게이트 — [카카오페이 SDD](https://tech.kakaopay.com/post/ifkakao-agentic-coding/)
- HITL/가드레일: 비가역·고위험·모호한 결정에만 질문 — [toss.tech](https://toss.tech/article/44539); 제출 전 전송내용 표시 후 승인, 배포는 요청 시만, OAuth only — [apps-in-toss-harness](https://github.com/toss/apps-in-toss-harness)
- 보안/거버넌스: GitHub App 단기 PR 스코프 토큰, Bedrock 사내 프록시, 중앙 정책 레포 — [LY](https://techblog.lycorp.co.jp/ko/building-ai-code-review-platform-with-claude-code-action); 플러그인 개별 버전·변경 공지 — [카카오페이](https://tech.kakaopay.com/post/ai-agent-2/)
- 실패 모드 모음: 파이프로 exit code 은폐, 공유 워킹트리 파괴(`git stash`), hook 포맷 오류로 무동작, 검증 과장 — [카카오페이](https://tech.kakaopay.com/post/ai-agent-2/); Lost in the Middle, 학습데이터 편향 회귀, 지시 희석, 오류 누적 — [컬리](https://helloworld.kurly.com/blog/vibe-coding-with-claude-code/); 컨텍스트 낭비·환각·응답 중단 — [우아한형제들](https://techblog.woowahan.com/26177/); constitution 무시·spec→tasks 누락·토큰 비용 — [카카오페이 SDD](https://tech.kakaopay.com/post/ifkakao-agentic-coding/); 업스트림 액션의 fork PR 가정, 사용자 입력이 출력 포맷 덮어씀 — [LY](https://techblog.lycorp.co.jp/ko/building-ai-code-review-platform-with-claude-code-action)
- 모델 향상에 따른 하네스 축소(21→11 에이전트, 29→17 스킬): "모델 한계 가정"을 담은 컴포넌트는 낡는다 — [카카오페이](https://tech.kakaopay.com/post/ai-agent-2/)
- 비용 경고: 단일 에이전트 루프 수만~수십만 토큰, 오케스트레이터+전문가 3명 플릿 루프 50만~200만 토큰 추정 — [BirdSpring](https://birdspring.com/loop-engineering-facts-2026) [개인, 추정치]
- 공급망 보안 참고(한국 외): Cursor 마켓플레이스에 탈취 키로 악성 Claude Code 확장이 약 11시간 배포, 약 5,000명 다운로드 — [BleepingComputer](https://www.bleepingcomputer.com/news/security/cursor-confirms-security-breach-led-to-malicious-claude-code-extension/)
- Claude Code가 2.1.277(2026-09-18)부터 CLAUDE.md가 없으면 AGENTS.md를 읽는다는 보도(여러 2차 자료, 공식 릴리스노트 미확인) — [Hacker News](https://news.ycombinator.com/item?id=49760187)

### Inferences
- loop-harness에 반영할 "한국 실무형" 기본값: 루트 지침 ≤150행, 경로 스코프 규칙, 검증 게이트는 exit code 기반 + 파이프 금지/`pipefail`, 세션별 worktree 격리, Stop 게이트, 비가역 액션 HITL, 하네스 변경 A/B 지표(성공률·토큰·재작업률), 중앙 정책 + 프로젝트 오버레이 병합(LY 패턴).
- 정량 평가(eval)를 공개한 한국 사례는 거의 없음(LY 사용량, 우아한형제들 바이트 절감 정도) → 평가 파이프라인은 차별화 포인트.

### Gaps
- 한국 기업의 정량적 eval(성공률, 오탐률, 비용) 공개 자료 부재.
- 금융권(카카오뱅크·토스뱅크 등)의 망분리/규제 환경에서의 코딩 에이전트 거버넌스 공개 자료 미확인.

## Q4. 한국 커뮤니티 용어: 루프 엔지니어링 / 하네스 엔지니어링 / 컨텍스트 엔지니어링

### Takeaway
2026년 한국 커뮤니티에서는 프롬프트 → 컨텍스트 → 하네스 → 루프 엔지니어링의 계층 구조로 설명하는 것이 일반적이다. "하네스"는 단일 에이전트의 실행 환경(도구·권한·검증·제어), "루프"는 그 하네스를 트리거/스케줄로 반복 운영하며 검증·기록·다음 작업 결정까지 설계하는 상위 층으로 쓰인다.

### Cited Findings
- SK AX(2026-08-28): 프롬프트 엔지니어링=질문 다듬기, 컨텍스트 엔지니어링=판단에 필요한 정보 구조화, 하네스 엔지니어링=단일 에이전트의 안전한 실행 환경(도구 호출·오류 처리·통제), 루프 엔지니어링=하네스 위에서 스케줄/트리거로 실행·서브에이전트 호출·결과 재투입. 비유 "하네스는 작업장, 루프는 작업장의 운영 방식". 구성요소: Automations, Worktrees, Skills, Plugins/MCP, Subagents(builder/verifier 분리), State(대화 밖 마크다운/이슈보드). 용어 기원은 Addy Osmani(2026-06)로 소개. — [SK AX](https://www.skax.co.kr/insight/trend/3840)
- 루프 명세 = 트리거·목표·검증 단계·중단 규칙·메모리로 구성된 재사용 아티팩트, Claude Code/Codex 같은 하네스에 전달; 루프 엔지니어링은 프롬프트/컨텍스트 엔지니어링을 대체하지 않고 "감싼다(wraps)" — [Appen Korea](https://kr.appen.com/blog/what-is-loop-engineering); [PyTorchKR](https://discuss.pytorch.kr/t/loop-engineering/10796)
- 비판적 시각: "새로운 마법이라기보다 오래된 패턴(ReAct)에 붙은 새 이름이자 무게중심이 운영 설계로 옮겨가는 흐름" — [BirdSpring](https://birdspring.com/loop-engineering-facts-2026) [개인]
- 관련 논문(한국어 요약): 「코딩 에이전트 과보호는 이제 그만: 단계별 프롬프팅을 대체하는 루프 엔지니어링」 — [alphaxiv](https://www.alphaxiv.org/ko/abs/2607.00038)
- "하네스"의 어원 '마구(馬具)' 설명이 기업 글에 공통 등장 — [toss.tech](https://toss.tech/article/44539); 카카오페이는 Harness를 제어·감시·개선 역할로 정의 — [카카오페이](https://tech.kakaopay.com/post/ai-agent-1/); 우아한형제들은 "AI가 길을 잃지 않도록 외부 통제 환경 구축"으로 정의하고 "컨텍스트 최적화"라는 표현 사용(컨텍스트 엔지니어링 용어는 미사용) — [우아한형제들](https://techblog.woowahan.com/26177/)
- 하네스 엔지니어링은 OpenAI Ryan Lopopolo 블로그에서 먼저 제시됐다는 커뮤니티 설명 — [Substack 랄프톤 후기](https://briandwjang.substack.com/p/8d3) [개인]
- 개인 정리: 하네스를 Context/권한/검증/도구/서브에이전트/디버깅 등 8가지 유형으로 분류 — [velog](https://velog.io/@leesh970930/Claude-Code-Harness-Engineering-%EA%B0%95%EC%9D%98-%EC%9A%94%EC%95%BD) [개인]; CLAUDE.md→프로젝트 파이프라인→skill-creator 자동 개선 구축기 — [velog](https://velog.io/@cjkangme/%ED%94%84%EB%A1%9C%EB%8D%95%EC%85%98-%EC%BD%94%EB%93%9C%EB%A5%BC-%EB%A7%8C%EB%93%9C%EB%8A%94-Claude-Code-%ED%95%98%EB%84%A4%EC%8A%A4-%EA%B5%AC%EC%B6%95%EA%B8%B0) [개인]
- 출판/교육 시장 형성: 한빛+ 『하네스 엔지니어링 with 클로드 코드』 — [한빛+](https://www.hanbit.co.kr/books/%ED%95%98%EB%84%A4%EC%8A%A4-%EC%97%94%EC%A7%80%EB%8B%88%EC%96%B4%EB%A7%81-with-%ED%81%B4%EB%A1%9C%EB%93%9C-%EC%BD%94%EB%93%9C?code=B2817272480); 패스트캠퍼스 「하네스 엔지니어링: Claude code · Codex 완벽 가이드」 — [패스트캠퍼스](https://fastcampus.co.kr/data_online_harness)

### Inferences
- "하네스 엔지니어링"은 2026년 상반기에 기업 기술블로그 제목에까지 들어간 주류 용어(우아한형제들 4월, 카카오페이 9·11월), "루프 엔지니어링"은 2026년 하반기 등장한 신조어로 아직 기업 실무 글보다는 컨설팅/미디어/커뮤니티 중심. loop-harness 소개 시 "하네스(단일 실행 환경) + 루프(반복 운영)" 2층 구조로 설명하면 한국 독자에게 잘 통할 것.

### Gaps
- 루프 엔지니어링 기원(Addy Osmani 2026-06 글)은 SK AX의 2차 언급만 확인, 원문 미확인.
- 한국 기업 기술블로그 중 "루프 엔지니어링"을 제목으로 쓴 공식 실무 글은 찾지 못함.
