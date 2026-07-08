# OpenWiki Skill — 설계

날짜: 2026-07-07
상태: v2 — 설계 리뷰 후 verbatim 포팅 전략으로 개정 (커뮤니티 포트 비교 리뷰 반영)

## 목표

[langchain-ai/openwiki](https://github.com/langchain-ai/openwiki)(MIT)의 워크플로우를 Claude Code / Codex용 agent skill로 재구성한다.

OpenWiki의 제품 가치는 사실상 시스템 프롬프트(`src/agent/prompt.ts`)와 런타임 보조 로직(`src/agent/utils.ts`: git 증거 수집, 콘텐츠 스냅샷, no-op 판단, 메타데이터)에 모여 있다. 나머지는 LLM 배관이다(provider 클라이언트, deepagents 런타임, SQLite 체크포인터, 스트림 파싱, 자격증명 온보딩). 이 skill은 앞의 것을 이식하고 뒤의 것을 전부 제거한다 — API key 불필요, 별도 런타임 불필요.

## 포팅 전략: verbatim 이식

v1은 upstream 규칙을 요약·재구성했으나, 리뷰에서 요약 과정의 규칙 유실(스냅샷 no-op, 추가 사용자 지시 처리, non-git 추론 등)이 확인되어 **verbatim 이식**으로 전환한다:

- `src/agent/prompt.ts`의 시스템 프롬프트 전문을 SKILL.md에 그대로 싣는다. 하네스 차이만 `[adapted]` 마커로 표시한다(가상 경로→실제 경로·네이티브 도구, deepagents task tool→하네스 서브에이전트, CLI reference 섹션·chat 모드는 `[omitted]`).
- `utils.ts`의 런타임 로직은 기계적 스텝으로 감싼다: git 증거 수집(Step 1), 콘텐츠 스냅샷 해시(Step 2), 프롬프트 실행(Step 3), 해시 재계산+메타데이터(Step 4).
- 효과: 동작 충실도 확보 + **upstream sync가 라인 diff 적용 수준으로 단순화**(요약본은 upstream 변경 때마다 재해석 필요).

## 비목표

- 독립 CLI 없음, LLM API 직접 호출 없음, provider/모델 설정 없음.
- 대화형 chat 모드 없음 — 호스트 에이전트가 이미 대화형이다. (위키 기반 Q&A는 별도 `openwiki-ask` skill이 담당.)
- Claude Code plugin 패키징(.claude-plugin/커맨드) 없음, Codex 전용 변형 파일 없음 — 단일 SKILL.md에 하네스 노트로 커버 (결정: 이중 유지보수 방지).
- 자동화 테스트 없음. 검증은 아래 수동 시나리오로 한다.

## 레포 구조

```
openwiki-skill/
├── README.md                    # 설치(npx skills add JHSeo-git/openwiki-skill), 사용법, attribution
├── LICENSE                      # MIT (표준 GitHub 템플릿)
├── UPSTREAM.md                  # upstream pin(커밋 SHA) + 파일 매핑 + 동기화 절차
├── CHANGELOG.md                 # 릴리스/변경 기록 (upstream sync 반영 내역 포함)
├── docs/                        # 설계/계획 문서 (superpowers)
└── skills/
    ├── openwiki/
    │   ├── SKILL.md             # 생성/유지보수 본체: verbatim 프롬프트 + 기계적 4단계
    │   └── references/
    │       └── automation.md    # keyless 자동화 + CI 템플릿 + 권한 allowlist
    └── openwiki-ask/
        └── SKILL.md             # 위키를 1차 소스로 질문에 답하는 소비측 skill
```

skill을 `skills/` 하위에 두는 이유: 레포 루트의 `openwiki/` 디렉토리명이 위키 출력 규약과 충돌하는 것을 방지하고, 다중 skill(openwiki, openwiki-ask)을 수용한다.

## skills/openwiki/SKILL.md

Frontmatter: name `openwiki`, description은 짧은 트리거 문구(레포 위키 문서를 `openwiki/`에 생성/유지보수, init/update 자동 감지).

**모드 해석**
- 사용자가 명시(초기화/업데이트)하면 그 모드.
- 아니면 자동 감지: `openwiki/quickstart.md` 있으면 update, 없으면 init. `openwiki/`만 있고 quickstart가 없으면 init이되 기존 파일 보존.
- 그 외 사용자 텍스트("API 라우트 위주로" 등)는 **추가 사용자 지시**로 run에 append (upstream `appendUserMessage` 이식).

**모델 티어 노트**: upstream 기본 모델군은 프론티어 코딩 모델 — 소형/고속 티어로 돌리지 말라는 한 줄 경고.

**Step 1 — git 증거 수집** (쓰기 전에; `git --no-pager` 사용, 전부 읽기 전용)
- `openwiki/.last-update.json` 읽기 → `status --short`, `rev-parse HEAD`, `diff --name-status HEAD` + 모드별 히스토리(`log <gitHead>..HEAD` / `--since <updatedAt>` / `--max-count=20`, `--name-status --oneline`).
- **early no-op** (update, 사용자 추가 지시 없을 때 — upstream `getUpdateNoopStatus` 이식): 워크트리가 깨끗하고 `gitHead` 이후 커밋이 없거나 `openwiki/`만 건드렸으면 → "wiki is already current" 보고 후 종료.
- git 레포가 아니면: 파일 타임스탬프·소스 조사·기존 문서로 변경을 **추론**해 진행 (upstream 동일).

**Step 2 — 콘텐츠 스냅샷** (작업 전; upstream `createOpenWikiContentSnapshot` 이식)
- `openwiki/` 트리(`.last-update.json` 제외)의 경로+내용 해시. macOS/Linux 겸용 `shasum -a 256` 기본, `sha256sum` 대체 노트.

**Step 3 — 시스템 프롬프트 (verbatim)**
- `src/agent/prompt.ts` @ pin 그대로: 역할, run discipline, subagent discipline, planning discipline(`_plan.md`), git discipline, existing docs discipline, root instruction files(+ 정확한 `## OpenWiki` 템플릿 — upstream과 byte 일치), security rules, documentation goals, section quality rules, required structure, 모드별 지시(init/update).
- `[adapted]`: 도구·경로, 서브에이전트(Claude Code Task tool / 서브에이전트 없는 하네스는 순차+컨텍스트 절약), 메타데이터 기록 주체(CLI→Step 4), 루트 지침 파일 신규 생성 시 `@AGENTS.md`를 import하는 CLAUDE.md 동반 생성(Claude Code는 CLAUDE.md만 읽음 — 공식 memory 문서의 권장 패턴; import형 CLAUDE.md는 update 시 semantically current로 간주).
- `[omitted]`: OpenWiki CLI reference 섹션, chat 모드.

**Step 4 — 메타데이터 기록** (작업 후)
- 스냅샷 재계산. 불변 → no-op 선언, 아무것도 안 씀. 변경 → `openwiki/.last-update.json` 기록: `{ updatedAt, command, gitHead, model }` (upstream `writeLastUpdateMetadata` 스키마). `updatedAt`/`gitHead`는 셸에서 실측(`date -u`, `git rev-parse HEAD`), 추측 금지. `model`은 호스트 모델 id, 모르면 `claude-code`/`codex`.

**실행 user prompt**: upstream `createUserPrompt`의 init/update 블록 verbatim + `Additional user instruction:` append 규칙.

**자동화 라우팅**: 주기 실행 요청 시 `references/automation.md` 참조.

## skills/openwiki/references/automation.md

1. **Keyless (구독 인증) 옵션** — 사용자 목표("API key 없이")의 1순위 답:
   - 로컬 스케줄(cron/OS 스케줄러)에서 `claude -p "use the openwiki skill to update..."` — 구독 로그인 사용, key 불필요. 스코프드 권한으로 무인 실행.
   - Claude Code cloud routine(`/schedule`) — Anthropic 인프라에서 구독으로 실행, fresh clone이므로 커밋+푸시 포함해야 함.
   - in-session 반복(/loop 등) — 세션 한정, 시험용.
2. **스코프드 권한 allowlist** — headless용 `.claude/settings.json` 예시: git 읽기 명령(`--no-pager status/rev-parse/log/diff/show/blame`) + `find`/`shasum`/`rg`/`date` + Write/Edit은 `openwiki/**`·`AGENTS.md`·`CLAUDE.md`만. `--dangerously-skip-permissions`의 안전한 대안.
3. **CI (key 필요)**: GitHub Actions cron → skill 설치(레포 클론) → `claude -p` → `openwiki/` 변경 시 PR (upstream examples 각색). GitLab 변형. `ANTHROPIC_API_KEY` 또는 `claude setup-token` 토큰 필요 명시.
4. **Codex headless** 노트: `codex exec` 사용 (플래그는 구현 시 `codex exec --help`로 실측 확인).

## skills/openwiki-ask/SKILL.md

위키 소비측 (~30줄): 레포 질문에 위키를 1차 소스로 답한다.
- `openwiki/` 없으면: 그 사실을 알리고 openwiki skill 실행 제안(소스로 답은 가능하되 명시).
- `quickstart.md`에서 시작 → 링크 추적 + `openwiki/` grep으로 canonical 페이지 탐색 → **페이지를 인용**해 답변, 인라인 소스 참조(`src/...`)를 함께 노출.
- 위키가 stale하거나 질문을 못 덮으면: 소스로 검증 후 답하고 update 실행을 제안.

## Upstream 동기화

발상은 openwiki 자신의 `.last-update.json` 패턴과 동일 — 이 레포는 upstream의 git head를 추적한다.

**UPSTREAM.md** (레포 루트, 파일 1개):
- Pin: upstream 레포 URL, 파생 기준 커밋 SHA, 날짜. 최초 pin: `7d355379423172049308ba166ec2eff02c1c2e7d` (2026-07-06).
- 파일 매핑 — 감시 대상을 좁혀 배관 코드 diff 노이즈를 걸러낸다:

| upstream | skill 파일 |
|---|---|
| `src/agent/prompt.ts` | `skills/openwiki/SKILL.md` Step 3 + user prompt 블록 (라인 대응) |
| `src/agent/utils.ts` (git 증거·스냅샷·no-op·메타데이터) | `skills/openwiki/SKILL.md` Steps 1, 2, 4 |
| `src/constants.ts` (경로 규약) | `skills/openwiki/SKILL.md` |
| `examples/*.yml` | `skills/openwiki/references/automation.md` |

- 동기화 절차: oss 클론 갱신(shallow면 `--unshallow`) → `git log <pinned>..HEAD --oneline -- <매핑 경로들>` → 변경 없으면 no-op / 있으면 diff를 매핑 파일에 적용 → pin 갱신 + CHANGELOG 기록. **Step 3이 prompt.ts와 라인 대응이므로 sync는 사실상 diff 적용이다.**
- 감지는 순수 git diff라 LLM 불필요. CI 자동 감지는 두지 않음(최소 구조); 필요 시 주간 cron→issue를 나중에 추가 가능.

## 하네스 호환성

단일 SKILL.md. `[adapted]` 마커와 서브에이전트 조항에 하네스 분기(Claude Code: Task tool / 서브에이전트 없는 하네스: 순차 + 절약적 읽기)를 내장한다. Codex는 `npx skills` 설치로 동일 파일을 소비하고, headless는 automation.md의 `codex exec` 노트를 따른다.

## 검증 시나리오 (수동)

1. 작은 fixture 레포에 init → `openwiki/` 생성: quickstart 진입점, 페이지 예산 준수, AGENTS.md 섹션 byte 일치, `_plan.md` 삭제, 메타데이터 `command:init`.
2. 소스 변경 커밋 후 update → 외과적 수정(diff budget 준수), 메타데이터 `command:update` 갱신.
3. 변경 없이 update → early no-op 또는 스냅샷 불변 → "already current" 보고, 파일 churn 없음(기계적으로 확인 가능).
4. openwiki-ask에 fixture 질문 → quickstart/페이지 인용 + 인라인 소스 참조 노출 확인.

## Attribution & 라이선스

LICENSE는 표준 GitHub MIT 템플릿 그대로. 출처 표기는 README의 attribution 섹션에서 langchain-ai/openwiki(MIT)를 명시한다. 파생 내용: 시스템 프롬프트·user prompt(verbatim, adapted 마커), 스냅샷/no-op/메타데이터 로직(utils.ts 이식), AGENTS.md 섹션 템플릿(verbatim), CI 예제(각색).

참고한 커뮤니티 포트: jatinmayekar/openwiki-for-claude-code(MIT — ask skill·keyless 자동화 아이디어), SoulKyu/openwiki-cc(라이선스 없음 — verbatim 전략 참고만, 표현 복사 없음). 스냅샷·no-op 등 로직은 모두 upstream에서 직접 이식한다.
