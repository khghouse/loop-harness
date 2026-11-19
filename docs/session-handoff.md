# Session Handoff

> 다음 세션이 이어서 작업하기 위한 노트. 세션이 끝날 때마다 **덮어쓴다**(누적하지 않음). 이전 내용은 git 이력에서 확인한다.
>
> 마지막 갱신: 2026-11-19 · 클라우드 세션 · 브랜치 `ccr-09b08acb-khnc2j`

## 현재 상태

- 업계 표준 리서치 완료: `research_notes/에이전트 루프 하네스 업계 표준/`(6개), 종합 보고서 `reports/에이전트 루프 하네스 업계 표준.md`
- 진행 방향: 백엔드(Go)/프론트엔드(Next.js)는 보류. 하네스/컨텍스트 층, 그중 `docs/`부터 구축
- 샘플 기획서를 받아 실제 프로젝트처럼 돌려 보면서 하네스를 다듬을 예정 (샘플 기획서는 아직 받지 않음)
- caveman, ponytail 플러그인을 `.claude/settings.json`으로 시험 적용 중. 새 클라우드 컨테이너에서 `claude plugin list`가 비어 있었고 모드 지시문(CAVEMAN/PONYTAIL MODE ACTIVE)도 컨텍스트에 없었다. 즉 settings.json만으로는 자동 설치되지 않았다.
- 같은 세션에서 수동 설치(`claude plugin marketplace add juliusbrussee/caveman`, `dietrichgebert/ponytail` 후 `claude plugin install caveman@caveman`, `ponytail@ponytail`)는 성공해 둘 다 enabled로 나왔다. 단, 지시문은 세션 시작 시 주입되므로 그 세션에서는 적용 여부를 확인하지 못했다.

## 확정된 사항

`docs/decisions.md`는 구조가 정해지지 않아 아직 만들지 않았다. 그때까지 확정 사항은 여기에 둔다.

- 원본 기획서는 `docs/references/`에 둔다.
- 원본과 변환본은 같은 파일명에 확장자만 다르게 한다. 기획서 하나당 아래 3개만 둔다(디자인 시안 제외).
  ```
  docs/references/
  ├── 회원가입_기획서_v1.0.pdf      # 원본
  ├── 회원가입_기획서_v1.0.md       # 변환본 (에이전트가 읽는 것)
  └── 회원가입_기획서_v1.0.assets/  # 해당 PDF 안의 이미지(와이어프레임, 플로우 등)만
  ```
- 세션 인수인계 파일은 `docs/session-handoff.md`(이 파일).

## 다음 할 일

1. 새 세션에서 `claude plugin list`와 모드 지시문 유무를 다시 확인. 목록이 비어 있으면 setup script에 위 설치 명령을 넣는 방식으로 전환(ponytail은 userConfig 4개 미설정, caveman은 yarn/pnpm 락파일 패키지 미설치 안내가 있었으니 동작도 함께 확인)
2. 샘플 기획서 PDF를 받으면 `docs/references/`에 넣고 변환 시험
3. `docs/` 나머지 구조 확정: `specs/`, `plans/active|completed/`, `decisions.md` 형식

## 미정 / 보류

- 변환본 md 상단에 출처 정보 블록(frontmatter)을 넣을지: 지켜보기로 함
- PDF → md 변환 담당(사람 / Claude): 미정. 제안은 markitdown 초안 → `pdftoppm` 페이지 이미지 → Claude가 구조 복원 → 사람 검수
- 문서 언어(한국어 / 영어), AGENTS.md 작성, hook 구성, 샘플 프로젝트 코드 위치(이 저장소 / 별도 저장소)

## 주의

- markitdown 테스트 결과: 한글은 정상이지만 제목 계층, 표 구조, 페이지 구분이 사라지고 이미지는 추출되지 않음. 테스트한 것은 직접 만든 PDF이므로 실제 기획서로 다시 확인할 것.
- markitdown과 두 플러그인의 사용자 범위 설치는 이전 클라우드 컨테이너에만 있었다. 새 컨테이너에는 없으므로 필요하면 다시 설치한다. 플러그인은 `.claude/settings.json`만으로는 자동 설치되지 않음을 확인했다(위 "현재 상태" 참고).
- `.claude/settings.json`의 플러그인 설정은 시험용이다. 되돌리려면 파일을 삭제한다. 이 브랜치를 main에 병합하면 main에서도 켜진다.
