# OpenWiki Skill — 설계

날짜: 2026-07-07
상태: 설계 리뷰 승인됨

## 목표

[langchain-ai/openwiki](https://github.com/langchain-ai/openwiki)(MIT)의 워크플로우를 Claude Code / Codex용 agent skill로 재구성한다.

OpenWiki의 제품 가치는 사실상 시스템 프롬프트(`src/agent/prompt.ts`) 하나에 모여 있다: 탐색 규율, 문서 품질 규칙, init/update 워크플로우, git 증거(evidence) 수집, 메타데이터 관리. 나머지 코드는 LLM 배관이다(provider 클라이언트, deepagents 런타임, SQLite 체크포인터, 스트림 파싱, 자격증명 온보딩). 이 skill은 규칙들을 SKILL.md 패키지로 이식해 호스트 에이전트가 자기 도구로 직접 실행하게 하고, 배관은 전부 제거한다 — API key 불필요, 별도 런타임 불필요.

## 비목표

- 독립 CLI 없음, LLM API 직접 호출 없음, provider/모델 설정 없음.
- 대화형 chat 모드 없음 — 호스트 에이전트가 이미 대화형이다.
- 자동화 테스트 없음. 검증은 아래 수동 시나리오로 한다.

## 레포 구조

```
openwiki-skill/
├── README.md                  # 설치(npx skills add JHSeo-git/openwiki-skill), 사용법, attribution
├── LICENSE                    # MIT (표준 GitHub 템플릿)
├── UPSTREAM.md                # upstream pin(커밋 SHA) + 파일 매핑 + 동기화 절차
├── CHANGELOG.md               # 릴리스/변경 기록 (upstream sync 반영 내역 포함)
├── SKILL.md                   # 코어 skill
└── references/
    ├── init.md                # init 모드 규칙
    ├── update.md              # update 모드 규칙
    ├── agents-md-section.md   # AGENTS.md/CLAUDE.md 고정 템플릿 + 삽입 규칙
    └── ci-examples.md         # 주기적 update용 CI 템플릿
```

## SKILL.md (코어)

- Frontmatter: name은 `openwiki`, description은 짧은 트리거 문구(레포 위키 문서를 `openwiki/`에 생성/유지보수).
- 모드 자동판단: `openwiki/quickstart.md` 있으면 → update, 없으면 → init. 사용자가 명시하면 그에 따른다. 엣지 케이스: `openwiki/`는 있는데 `quickstart.md`가 없으면 → init으로 취급하되 기존 파일은 보존한다.

공통 규율 (upstream 시스템 프롬프트에서 이식):

- 표적 탐색만: 레포 트리, package/config 파일, README류, entrypoint, 라우팅, 스키마 파일, 도메인별 대표 파일. 전체 파일을 다 읽지 않는다.
- 하네스가 지원하면 읽기전용 서브에이전트 1-4개로 병렬 리서치(Claude Code: Task tool). 종합과 모든 쓰기는 메인 에이전트가 담당한다.
- 기존 문서(README, docs/, 런북, SKILL.md 파일)를 1차 소스로 취급. 문서와 소스가 충돌하면 stale 문서임을 지적하고 소스 증거를 우선한다.
- secret, 자격증명, `.env` 파일은 읽지도 문서화하지도 않는다.
- 쓰기는 `openwiki/` 아래에만. 유일한 예외는 루트 `AGENTS.md` / `CLAUDE.md`이며, 참조 섹션만 수정한다.
- 코드가 왜 존재하는지를 설명한다. 무엇이 들어있는지만 나열하지 않는다. 문서 독자는 사람 + 미래 에이전트.
- 탐색 후 임시 플랜 파일 `openwiki/_plan.md` 작성, 완료 전 삭제(하네스 플래닝 도구로 대체 가능).

Git 증거 (인라인 명령 블록):

```
git status --short
git rev-parse HEAD
# 이전 gitHead가 있는 update:
git log <lastHead>..HEAD --name-status --oneline
#   fallback: git log --since <updatedAt> --name-status --oneline
#   fallback: git log --max-count=20 --name-status --oneline
git diff --name-status HEAD
```

git 레포가 아니면: 증거 수집을 생략하고 파일시스템만으로 진행한다.

- 메타데이터, upstream 호환: `openwiki/.last-update.json` — `{ updatedAt, command, gitHead, model }`. `model`은 알 수 있으면 호스트 모델 id, 아니면 하네스 이름(`claude-code` / `codex`). 위키 콘텐츠가 실제로 바뀐 경우에만 기록한다.
- no-op 규칙 (update): `lastHead` 이후 커밋이 `openwiki/`만 건드렸거나(또는 없고) 워크트리가 깨끗하면 → "wiki is already current" 보고 후 아무것도 쓰지 않는다.
- 라우팅: init → `references/init.md` 읽기; update → `references/update.md`; 루트 지침 파일 수정 시 → `references/agents-md-section.md`; CI 설정 요청 → `references/ci-examples.md`.

## references/init.md

- 인벤토리 순서: 기존 문서 → entrypoint → package/config → 주요 도메인 디렉토리 → 테스트/eval → 데이터/스키마 → 운영 스크립트.
- "왜"는 최근 git 히스토리로: 고신호 파일에 표적 `git log` / `git blame` / `git show`.
- `quickstart.md` 먼저(고수준 개요 + 모든 섹션 링크), 이후 섹션 디렉토리(`architecture/`, `workflows/`, `domain/`, `operations/` 등 레포에 맞게).
- 예산: 최초 실행은 최대 8페이지. 작은 레포(주요 소스 ~10개 이하)는 quickstart + 1-2페이지.
- 품질 규칙: thin page/스텁 디렉토리 금지; 개념당 canonical 페이지 1곳; 새 디렉토리보다 넓은 페이지 내 heading 우선; 마무리 전 트리 리뷰로 저가치 페이지 병합/제거; 커밋 해시 목록을 문서에 남기지 않는다.

## references/update.md

- `.last-update.json` 읽기 → git 증거 수집 → docs impact plan 작성(소스 변경 → 영향받는 문서 → 필요한 수정 → 이유) → 외과적 수정만.
- diff budget: 변경된 소스 파일 5개 미만 → 위키 페이지 최대 1-2개 수정; 3페이지 초과 수정은 강한 근거 필요.
- 포맷팅-only 수정 금지; source map이나 "주의사항" 섹션은 실질적으로 틀렸을 때만 갱신; quickstart는 최상위 동작/설정/내비게이션이 바뀐 경우에만.
- 매 update 실행마다 루트 `AGENTS.md`/`CLAUDE.md` 참조 섹션의 의미적 stale 여부를 확인한다.
- no-op도 유효한 결과다.

## references/agents-md-section.md

- upstream `## OpenWiki` 섹션 템플릿을 그대로(verbatim) 사용 — upstream openwiki CLI로 만든 레포와 상호운용 유지.
- 규칙: 루트 파일만 대상; `AGENTS.md`와 `CLAUDE.md`가 둘 다 있으면 양쪽에 삽입; 둘 다 없으면 `AGENTS.md` 생성; 기존 섹션은 중복 추가 대신 교체; 의미적으로 stale할 때만 갱신; 무관한 포맷팅은 건드리지 않는다.

## references/ci-examples.md

- GitHub Actions cron 워크플로우: checkout → skill 설치 → headless 실행(`claude -p`) → `openwiki/` 변경 시 PR 생성. GitLab CI 변형 포함.
- 정직한 요구사항 고지: 로컬 대화형 사용은 API key 불필요하지만, CI 자동화는 `ANTHROPIC_API_KEY` 또는 `claude setup-token` OAuth 토큰이 필요하다.
- upstream `examples/openwiki-update.yml`을 참고해 작성.

## 하네스 호환성

skill 본문은 도구 중립 서술("파일 읽기/검색 도구", "지원 시 읽기전용 서브에이전트")을 유지하고, Claude Code 특화 내용은 괄호 힌트로만 덧붙인다. Codex는 `npx skills` 설치로 동일 파일을 소비한다.

## Upstream 동기화

이 skill은 upstream 스냅샷의 파생물이므로, upstream 변경을 추적할 최소 구조를 갖춘다. 발상은 openwiki 자신의 `.last-update.json` 패턴과 동일하다 — openwiki가 대상 레포의 git head를 추적하듯, 이 레포는 upstream의 git head를 추적한다.

**UPSTREAM.md** (레포 루트, 파일 1개):

- Pin: upstream 레포 URL, 파생 기준 커밋 SHA, 날짜. 최초 pin: `7d355379423172049308ba166ec2eff02c1c2e7d` (2026-07-06).
- 파일 매핑 테이블 — 감시 대상을 좁혀 배관 코드 diff 노이즈를 걸러낸다:

| upstream | skill 파일 |
|---|---|
| `src/agent/prompt.ts` | `SKILL.md`, `references/init.md`, `references/update.md`, `references/agents-md-section.md` |
| `src/agent/utils.ts` (git evidence·메타데이터·no-op) | `SKILL.md` |
| `src/constants.ts` (경로 규약) | `SKILL.md` |
| `examples/*.yml` | `references/ci-examples.md` |

- 동기화 절차 (에이전트가 요청받으면 그대로 수행):
  1. `git -C ~/Projects/oss/openwiki fetch` (shallow clone이면 `--unshallow` 먼저) 후 pull.
  2. `git log <pinned>..HEAD --oneline -- src/agent/prompt.ts src/agent/utils.ts src/constants.ts examples/` — 매핑된 파일의 변경만 확인.
  3. 변경 없음 → no-op 보고. 변경 있음 → diff 리뷰 후 매핑에 따라 해당 skill 파일에 이식.
  4. UPSTREAM.md pin 갱신 + CHANGELOG에 sync 내역 기록.

감지는 순수 git diff라 LLM이 필요 없고, 이식만 에이전트 작업이다. 별도 sync skill이나 CI 자동화는 두지 않는다(최소 구조). 필요해지면 주간 cron으로 2번 명령을 돌려 변경 감지 시 issue를 여는 GitHub Actions를 나중에 추가할 수 있다.

## 검증 시나리오 (수동)

1. 작은 실제 레포에 init 실행 → `openwiki/` 생성 확인: quickstart 진입점, 페이지 예산 준수, AGENTS.md 섹션 삽입, 메타데이터 기록.
2. 소스 변경 커밋 후 update 실행 → 외과적 수정, impact plan 준수, 메타데이터 갱신.
3. 변경 없이 update 실행 → no-op 보고, 파일 churn 없음.

## Attribution & 라이선스

LICENSE는 표준 GitHub MIT 템플릿 그대로 둔다. 출처 표기는 README의 attribution 섹션에서 langchain-ai/openwiki(MIT)를 명시하는 것으로 충분하다. 파생 내용: 프롬프트 규율 텍스트(재작성·압축), AGENTS.md 섹션 템플릿(verbatim), CI 예제(각색).
