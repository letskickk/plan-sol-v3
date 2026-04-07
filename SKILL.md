---
name: plan-sol-v3
preamble-tier: 3
version: 3.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /plan-sol-v3.
  Sol(오케스트레이터) → Planner(Opus) → Generator(Sonnet) → Evaluator(Bash-First) 피드백 루프.
  T2/T3/T5 = Bash CLI (0 LLM tokens). T1/T4 = LLM (판단 필요시만, T2+T5 통과 후).
  ALL >= 90, GM >= 92. 예외 없음.
triggers:
  - /plan-sol-v3
  - /sol-v3
tags:
  - planning
  - review
  - execution
  - orchestration
  - quality
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
  - AskUserQuestion
---

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
mkdir -p ~/.gstack/analytics 2>/dev/null || true
echo '{"skill":"plan-sol-v3","v":"3.0","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
```

## AskUserQuestion Format
1. **Re-ground:** Project, branch, current phase.
2. **Simplify:** Plain language.
3. **Recommend:** `RECOMMENDATION: Choose [X]` with `Completeness: X/10`.
4. **Options:** Lettered A/B/C.

## Completion Status Protocol
Report: **DONE** / **DONE_WITH_CONCERNS** / **BLOCKED** / **NEEDS_CONTEXT**.
Escalate after 3 failed attempts.

## Telemetry (run last)
```bash
_TEL_END=$(date +%s); _TEL_DUR=$(( _TEL_END - _TEL_START ))
~/.claude/skills/gstack/bin/gstack-telemetry-log --skill "plan-sol-v3" --duration "$_TEL_DUR" --outcome "OUTCOME" --session-id "$_SESSION_ID" 2>/dev/null &
```

---

# /plan-sol-v3 — Sol Backbone v3 (Bash-First Evaluator)

> "LLM이 해야만 하는 것에만 LLM을 쓴다. 나머지는 도구가 한다."

## PIPELINE

```
Phase 0: SOCRATES → Phase 1: CEO(Opus×2) → Phase 2: DESIGN(Sonnet, conditional)
→ Phase 3: ENG(Opus×2) + T1구조검증(Bash) → Phase 4: PLAN GATE
→ Phase 5: BUILD (Planner→Generator)
→ Phase 6: BASH EVAL → [T2+T5 통과시만] LLM EVAL → FIX LOOP
→ Phase 7: SHIP GATE
```

## SCORING

| Tier | Name | Pass | 평가 방법 |
|------|------|------|---------|
| T1 | Prompt Quality | >= 90 | LLM (haiku) — T2+T5 통과 후에만 |
| T2 | Code Quality | >= 90 | **Bash CLI** — tsc/console/any/circular |
| T3 | Visual UX | >= 90 | **Bash CLI** — rgba/hex/spacing grep |
| T4 | Functional | >= 90 | LLM (sonnet) — T2+T5 통과 후에만 |
| T5 | Deploy Safety | >= 90 | **Bash CLI** — secrets/AI-word/critical gate |

**ALL >= 90, GM >= 92. 예외 없음.**
**T2/T5 FAIL → T1/T4 LLM 호출 스킵. 기본이 안 된 코드에 판단 비용 낭비 없음.**

## MODEL ROUTING

| Task | Model | 이유 |
|------|-------|------|
| Phase 0 Socrates | user's model | 대화형 |
| Phase 1 CEO Primary + Challenger | opus × 2 | 전략 판단 |
| Phase 2 Design | sonnet | 패턴 매칭 |
| Phase 3 Eng Primary + Challenger | opus × 2 | 아키텍처 |
| Phase 5 Planner | opus | 빌드/수정 계획 수립 |
| Phase 5 Generator | sonnet | 코드 실행 |
| Phase 5 Info Gather | haiku | 파일 읽기 |
| Phase 6 T1 Prompt Quality | haiku | T2+T5 통과 후 |
| Phase 6 T4 Functional | sonnet | T2+T5 통과 후 |
| Phase 6 FIX Planner | opus | 수정 계획 |
| Breakthrough (11+) | opus | 메타 분석 |

**CRITICAL**: Subagents CANNOT spawn subagents. Main thread orchestrates ALL.

## GLOBAL RULES

### Checkpoint
`<!-- CHECKPOINT: phase=N status=complete timestamp=ISO -->`
Resume: plan file 체크포인트 읽기 → AskUserQuestion(Resume/Restart/Jump).

### Context Budget (핵심)
- Agent에 plan file 전체를 넘기지 않는다. 해당 Agent가 필요한 섹션만 추출해서 전달.
- Phase 6 Bash Evaluator: plan file 불필요. CLI 결과만으로 점수 계산.
- Phase 6 FIX Planner: plan file의 BUILD_PLAN + Bash 위반 목록만 전달.
- 2000 토큰 초과 시 Haiku로 요약 후 전달.

### Language Mirroring
Mirror user's language. Agent prompts: "Mirror the user's language."

### Section Skip List
Review 스킬 파일 로드 시 건너뛸 섹션:
Preamble, AskUserQuestion Format, Completion Status Protocol, Telemetry,
Completeness Principle, Search Before Building, Contributor Mode,
Step 0: Detect base branch, Prerequisite Skill Offer, Outside Voice.

### 6 Auto-Decision Principles
1. Completeness 2. Boil lakes 3. Pragmatic 4. DRY 5. Explicit over clever 6. Bias toward action

---

## PHASE 0: SOCRATES — Problem Discovery

### Rules
1. NEVER give direct answers 2. NEVER propose solutions 3. Guide through questions ONLY 4. Mirror language

### Step 0A: Context Gathering (Silent)
Read CLAUDE.md, TODOS.md, `git log --oneline -20`, referenced files.

### Step 0B: Opening Assessment
ONE question: "이 기능이 해결하려는 핵심 문제가 뭐라고 생각하세요?"

### Step 0C: Guided Discovery (3-7 rounds)

| Level | Type | Example |
|-------|------|---------|
| 1 | Clarifying | "그 결론에 도달한 근거가 뭔가요?" |
| 2 | Probing | "Y가 없으면 어떻게 되나요?" |
| 3 | Connecting | "이게 Z와 어떤 관계가 있나요?" |
| 4 | Counter | "만약 A가 아니라 B라면요?" |
| 5 | Hypothetical | "프로덕션에 나가면 어떤 문제가 생길까요?" |

"모르겠어요" → 더 작은 질문으로 쪼갬. 답 요구 시 → "한 단계만 더 생각해보죠."

### Step 0D: Problem Definition Lock

```
┌─────────────────────────────────────────┐
│          PROBLEM DEFINITION             │
├─────────────────────────────────────────┤
│ Problem:  [user's own words]            │
│ Who:      [affected users/stakeholders] │
│ Current:  [how it's solved now]         │
│ Pain:     [specific pain points]        │
│ Success:  [what "solved" looks like]    │
│ Not:      [explicitly out of scope]     │
└─────────────────────────────────────────┘
```

AskUserQuestion → A) 정확 B) 수정 C) 처음부터

### Step 0E: Plan Seed
Write plan file:
```markdown
# Plan: [Project Name]
## Problem Definition (Socratic Discovery)
[Problem Definition Card]
## Initial Scope
## Open Questions
## Constraints
## Prompt Design (T1 검증 대상)
- 페르소나 이름/성격:
- 축별 톤 분기:
- 지원 언어:
- system/user 역할 분리:
- AI 용어 금지 규칙:
- 행동 제안 의무:
## Design Tokens (T2/T3 검증 기준)
- COLORS: (primary, secondary, ...)
- SPACING: (xxs:2, xs:4, sm:8, md:16, lg:24, xl:32)
- FONTS: (headline, body)
```

`<!-- CHECKPOINT: phase=0 status=complete -->`

---

## PHASE 1: CEO REVIEW (Opus × 2)

### Step 1A: Load + Primary (Opus)
```bash
# 스킬 파일 로드 시도
cat ~/.claude/skills/plan-ceo-review/SKILL.md 2>/dev/null || echo "FALLBACK"
```

```
Agent(model: "opus", prompt: "
  CEO/founder reviewing a development plan.
  [loaded skill content, skip-list 적용 OR embedded fallback]

  PLAN PROBLEM DEFINITION: [Phase 0 Problem Definition Card만 — 전체 plan 아님]
  PLAN SCOPE: [Initial Scope 섹션만]

  Required: premise challenge, code leverage map, dream state diagram,
  alternatives table (2-3), scope decisions, error registry, failure modes, NOT in scope.
  6 principles (CEO: P1+P2). Decision log: | # | Decision | Principle | Rationale | Rejected |
  Mirror language.
")
```

### Step 1B: Challenger (Opus)
```
Agent(model: "opus", prompt: "
  Independent CEO. NO prior review.
  PLAN SCOPE: [scope 섹션만]
  Evaluate: 1)right problem? 2)premises? 3)6-month regret? 4)alternatives? 5)competitive risk?
  Adversarial. No compliments. Mirror language.
")
```

### Step 1C: Consensus + Checkpoint
Premise Gate: AskUserQuestion (전제 확인은 자동결정 불가).
```
=== PHASE 1 COMPLETE === Consensus: [X/6]
<!-- CHECKPOINT: phase=1 status=complete -->
<!-- COST: phase=1 model=opus calls=2 -->
```

---

## PHASE 2: DESIGN REVIEW (Sonnet, conditional)

Skip if <2 UI keywords (component/screen/form/button/modal/layout/dashboard/nav/dialog).

```
Agent(model: "sonnet", prompt: "
  Senior product designer. Rate 7 dimensions 0-10:
  1.Hierarchy 2.Specificity 3.Empty states 4.Interaction patterns
  5.Accessibility 6.Visual coherence 7.Performance

  PLAN SCOPE + DESIGN TOKENS: [해당 섹션만]
  CEO SUMMARY: [Phase 1 요약 max 300 tokens]
  Auto-decide structural. Defer taste. 6 principles (P5+P1). Mirror language.
")
```

```
<!-- CHECKPOINT: phase=2 status=complete -->
<!-- COST: phase=2 model=sonnet calls=1 -->
```

---

## PHASE 3: ENG REVIEW (Opus × 2) + T1 구조검증

### Step 3A: Primary Eng (Opus)
```
Agent(model: "opus", prompt: "
  Senior engineering manager.
  [loaded eng skill content OR embedded fallback]

  PLAN FULL: [plan file — Phase 3는 전체 필요]
  CEO SUMMARY: [max 400 tokens]
  DESIGN SUMMARY: [max 200 tokens or 'skipped']

  Required: architecture ASCII diagram, test diagram, test plan artifact (write to disk),
  failure modes, NOT in scope, what already exists.
  NEVER skip Test Review. 6 principles (P5+P3). Mirror language.
")
```

### Step 3B: Eng Challenger (Opus)
```
Agent(model: "opus", prompt: "
  Independent senior engineer. NO prior review.
  PLAN SCOPE + ARCHITECTURE: [해당 섹션만]
  Evaluate: architecture/edge cases 10x/missing tests/security/hidden complexity.
  Adversarial. Mirror language.
")
```

### Step 3C: T1 구조검증 (Bash)
Plan file의 Prompt Design 섹션을 Bash로 검증:

```bash
PLAN_FILE="[plan file path]"

check() { grep -q "$1" "$PLAN_FILE" 2>/dev/null && echo "PASS" || echo "FAIL: $2"; }

R1=$(check "페르소나\|persona" "페르소나 이름/성격")
R2=$(check "축별\|tone\|분기" "톤 분기")
R3=$(check "ko\|en\|ja\|zh\|언어" "지원 언어")
R4=$(check "system.*user\|역할 분리" "system/user 분리")
R5=$(check "AI 용어\|forbidden\|금지" "AI 용어 금지")
R6=$(check "행동 제안\|advice\|recommend" "행동 제안")
R7=$(check "COLORS:\|SPACING:\|FONTS:" "Design Tokens")

PASS_COUNT=$(echo "$R1 $R2 $R3 $R4 $R5 $R6 $R7" | grep -o "PASS" | wc -l)
[ "$PASS_COUNT" -ge 6 ] && echo "T1_STRUCT: PASS ($PASS_COUNT/7)" || echo "T1_STRUCT: FAIL ($PASS_COUNT/7)"
```

FAIL → 메인 스레드가 plan file 보완 후 재실행 (max 2회). Phase 4 진입 조건: PASS.

### Step 3D: Test Plan + Checkpoint
```bash
SLUG=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "project")
mkdir -p ~/.gstack/projects/$SLUG 2>/dev/null || true
# Write test plan → ~/.gstack/projects/$SLUG/test-plan-$(date +%Y%m%d).md
```

```
=== PHASE 3 COMPLETE === T1 Structure: PASS
<!-- CHECKPOINT: phase=3 status=complete -->
<!-- COST: phase=3 model=opus calls=2 -->
```

---

## PHASE 4: PLAN GATE

Pre-Gate: Problem Definition / Prompt Design / Design Tokens / CEO / Design / Eng / Test Plan / T1 PASS 모두 확인.

```
╔══════════════════════════════════════════════════╗
║        PLAN SOL V3 — REVIEW COMPLETE             ║
╠══════════════════════════════════════════════════╣
║  Phase 0: Problem [one-line] | Rounds [N]        ║
║  Phase 1: CEO Consensus [X/6]                    ║
║  Phase 2: Design [N]/10 (or SKIPPED)             ║
║  Phase 3: Eng [sound/concerns] | Test [path]     ║
║  T1 Structure: PASS                              ║
╚══════════════════════════════════════════════════╝
```

AskUserQuestion: A)Approve B)Override(taste) C)Interrogate D)Revise E)Reject F)Pause

`<!-- CHECKPOINT: phase=4 status=approved -->`

---

## PHASE 5: BUILD — Planner → Generator

### 역할
- **Sol**: 판단 없음. 전달 + 흐름 제어.
- **Planner (Opus)**: 뭘 어떻게 만들지, 뭘 어떻게 고칠지.
- **Generator (Sonnet)**: Planner 지시 그대로 실행. 설계 결정 없음.
- **검증**: Phase 5에서 하지 않음. Phase 6 Bash Evaluator가 담당.

### Step 5A: Info Gathering (Haiku)
```
Agent(model: "haiku", prompt: "
  Read files in eng review architecture + existing code map.
  Output CONTEXT BRIEF (max 2000 tokens): exports, patterns, integration points.
")
```

### Step 5B: Planner — BUILD_PLAN 수립 (Opus)
```
Agent(model: "opus", prompt: "
  You are the Planner. Create BUILD_PLAN for the Generator.

  PLAN SCOPE + DESIGN TOKENS: [해당 섹션만]
  CONTEXT BRIEF: [from 5A]

  Units (의존성 순서):
  Unit 1: config (theme.ts, types) | Unit 2: utils (calculators)
  Unit 3: services (GPT, auth, DB) | Unit 4: stores (state)
  Unit 5: components (new) | Unit 6: components (legacy copy+tokenize)
  Unit 7: screens | Unit 8: navigation + App.tsx | Unit 9: DB + Edge Functions

  For each Unit — files, Quality Rules, token mappings for legacy.
  Quality Rules MUST include:
  - COLORS 토큰만 (hex/rgba 금지)
  - SPACING 토큰만 (리터럴 숫자 금지, StyleSheet.hairlineWidth 예외)
  - any 타입 금지 (catch: unknown)
  - console은 if (__DEV__) 내부만
  - 순환참조 금지

  Output: BUILD_PLAN. Sol이 plan file에 저장.
")
```

Sol → BUILD_PLAN을 plan file에 저장.

### Step 5C: Generator (Sonnet) — Unit별 순차 실행
```
FOR each Unit (1→9):
  Agent(model: "sonnet", prompt: "
    You are the Generator. Execute Planner's instructions exactly.
    BUILD_PLAN Unit [N]: [해당 Unit 지시만]
    PREVIOUS UNITS SUMMARY: [이전 Unit 결과 요약, max 500 tokens]

    DO NOT make design decisions. Follow BUILD_PLAN exactly.
    List ALL files created/modified after completion.
    Mirror language.
  ")
```

### Step 5D: Checkpoint
```
=== PHASE 5 COMPLETE ===
<!-- CHECKPOINT: phase=5 status=complete -->
<!-- COST: phase=5 model=opus calls=1 model=sonnet calls=[N] model=haiku calls=1 -->
```

---

## PHASE 6: Bash-First Evaluator Loop

**루프 변수**: `iteration=1`, `consecutive_pass=0`

### Step 6A: ★ Bash Evaluator (0 LLM tokens)

프로젝트 루트에서 실행:

```bash
# ── T2: Code Quality ──────────────────────────────────────────────
TSC_ERRORS=$(npx tsc --noEmit 2>&1 | grep -c "error TS" 2>/dev/null || echo 0)

CONSOLE_UNGUARDED=$(grep -rn "console\." src/ --include="*.ts" --include="*.tsx" 2>/dev/null \
  | grep -v "if (__DEV__)" | grep -v "^\s*//" | wc -l | tr -d '[:space:]')

ANY_EXPLICIT=$(grep -rn ": any" src/ --include="*.ts" --include="*.tsx" 2>/dev/null \
  | grep -v "eslint-disable" | grep -v "^\s*//" | wc -l | tr -d '[:space:]')

CIRCULAR_COUNT=$(npx madge --circular --extensions ts,tsx src/ 2>/dev/null \
  | grep -c "Found" || echo 0)

T2_TSC=$(( TSC_ERRORS == 0 ? 100 : (100 - TSC_ERRORS * 5 < 0 ? 0 : 100 - TSC_ERRORS * 5) ))
T2_CON=$(( 100 - CONSOLE_UNGUARDED * 2 < 70 ? 70 : 100 - CONSOLE_UNGUARDED * 2 ))
T2_ANY=$(( 100 - ANY_EXPLICIT * 3 < 0 ? 0 : 100 - ANY_EXPLICIT * 3 ))
T2_CIR=$(( CIRCULAR_COUNT == 0 ? 100 : (100 - CIRCULAR_COUNT * 15 < 0 ? 0 : 100 - CIRCULAR_COUNT * 15) ))
T2_SCORE=$(( (T2_TSC + T2_CON + T2_ANY + T2_CIR) / 4 ))

# ── T3: Visual UX (mechanical) ────────────────────────────────────
RGBA_VIO=$(grep -rn "rgba(" src/ --include="*.ts" --include="*.tsx" 2>/dev/null \
  | grep -v "theme\.ts" | grep -v "^\s*//" | wc -l | tr -d '[:space:]')

HEX_VIO=$(grep -rn "#[0-9a-fA-F]\{3,8\}\b" src/ --include="*.ts" --include="*.tsx" 2>/dev/null \
  | grep -v "theme\.ts" | grep -v "^\s*//" | wc -l | tr -d '[:space:]')

SPACING_VIO=$(grep -rn "padding: [0-9]\+[,;]\|margin: [0-9]\+[,;]\|gap: [0-9]\+[,;]" src/ \
  --include="*.ts" --include="*.tsx" 2>/dev/null \
  | grep -v "theme\.ts" | grep -v "hairlineWidth" | grep -v "^\s*//" | wc -l | tr -d '[:space:]')

T3_RAW=$(( 100 - RGBA_VIO * 10 - HEX_VIO * 5 - SPACING_VIO * 5 ))
T3_MECH=$(( T3_RAW < 0 ? 0 : T3_RAW ))

# ── T5: Deploy Safety ─────────────────────────────────────────────
SECRETS=$(grep -rn "sk-[a-zA-Z0-9]\{20\}" src/ --include="*.ts" --include="*.tsx" 2>/dev/null \
  | grep -v "process\.env" | grep -v "^\s*//" | wc -l | tr -d '[:space:]')

AI_UI=$(grep -rn '"AI"\|'\''AI'\''\| AI[^a-zA-Z_]' src/ --include="*.tsx" 2>/dev/null \
  | grep -v "SafeAreaView\|AI_\|^\s*//" | wc -l | tr -d '[:space:]')

# T5 critical gate: tsc/circular/secrets 중 하나라도 → cap 50
if [ "${TSC_ERRORS:-0}" -gt 0 ] || [ "${CIRCULAR_COUNT:-0}" -gt 0 ] || [ "${SECRETS:-0}" -gt 0 ]; then
  T5_SCORE=50
else
  T5_RAW=$(( 100 - CONSOLE_UNGUARDED * 1 - ANY_EXPLICIT * 1 - AI_UI * 5 ))
  T5_SCORE=$(( T5_RAW < 0 ? 0 : T5_RAW ))
fi

# ── Output ────────────────────────────────────────────────────────
echo "=== BASH EVAL: Round $iteration ==="
echo "T2=$T2_SCORE [tsc=$TSC_ERRORS | console=$CONSOLE_UNGUARDED | any=$ANY_EXPLICIT | circular=$CIRCULAR_COUNT]"
echo "T3_mech=$T3_MECH [rgba=$RGBA_VIO | hex=$HEX_VIO | spacing=$SPACING_VIO]"
echo "T5=$T5_SCORE [secrets=$SECRETS | AI_UI=$AI_UI | critical=$([ $T5_SCORE -le 50 ] && echo YES || echo NO)]"
```

### Step 6B: LLM Evaluator — T2+T5 통과 후에만

```
IF T2_SCORE < 90 OR T5_SCORE < 90:
  # LLM 호출 없음. 바로 6D FAIL로.
  T1_SCORE = "BLOCKED (T2=$T2_SCORE T5=$T5_SCORE)"
  T3_SCORE = T3_MECH  # mechanical만
  T4_SCORE = "BLOCKED"
  GOTO Step 6D (FAIL)
```

T2+T5 모두 >= 90일 때만:

**T1 — Prompt Quality (Haiku)**
```
Agent(model: "haiku", prompt: "
  T1 PROMPT QUALITY — openai.ts (또는 GPT 서비스 파일) 읽어서 평가.

  Bash에서 이미 확인한 것 (재확인 불필요):
  - 'AI' 단어: Bash T5에서 카운트됨

  LLM만 판단 가능한 것:
  1. 페르소나 일관성 (이름/어조가 ko/en/ja/zh 전체에서 동일한가) → 0-100
  2. 어드바이스 구체성 (막연한 위로 vs 실행 가능한 행동 제안) → 0-100
  3. 도메인 데이터 활용 (사주/카드명 등 실제 데이터가 프롬프트에 있는가) → 0-100
  4. system/user 역할 분리 (API 호출에서 역할이 섞이지 않는가) → 0-100
  5. 4개 언어 지원 (ko/en/ja/zh prompt 분기가 있는가) → PASS/FAIL (-20 per missing)

  T1_SCORE = avg(1,2,3,4) + language_deduction. Clamped 0-100.
  Output: T1_SCORE: [N] | breakdown: [per check]
  Mirror language.
")
```

**T4 — Functional (Sonnet)**
```
Agent(model: "sonnet", prompt: "
  T4 FUNCTIONAL — Test Plan 기반 검증.
  TEST PLAN: [~/.gstack/projects/SLUG/test-plan-*.md 경로]

  85%: 계산 함수 정확성, 스크린 export, store CRUD, Edge Function 파싱
  15%: 엣지 케이스 (빈 입력, null, 경계값)

  각 테스트 항목: PASS / FAIL / SKIP (파일 없음)
  T4_SCORE = (PASS + SKIP*0.5) / total * 100. Clamped 0-100.
  Output: T4_SCORE: [N] | failed: [list]
  Mirror language.
")
```

**T3 최종 점수**:
T3_MECH >= 90이면 T3_SCORE = T3_MECH.
T3_MECH < 90이면 T3_SCORE = T3_MECH (LLM으로 올릴 수 없음 — 코드 수정만이 답).

### Step 6C: GM 계산

```
GM = T1*0.25 + T2*0.15 + T3*0.25 + T4*0.20 + T5*0.15

=== ROUND [N] RESULTS ===
T1=[N] T2=[N] T3=[N] T4=[N] T5=[N] GM=[N.N]
Status: [ALL>=90 AND GM>=92 → PASS | otherwise FAIL]
Violations: tsc=[N] console=[N] any=[N] circular=[N] rgba=[N] hex=[N] spacing=[N]
```

### Step 6D: PASS / FAIL 분기

**PASS:**
```
consecutive_pass += 1
IF consecutive_pass >= 3: BREAK → Phase 7
ELSE: iteration += 1, GOTO 6A
```

**FAIL:**
```
consecutive_pass = 0

IF iteration >= 11: BREAKTHROUGH MODE (Step 6E)

Sol → Planner(Opus):
  "FIX REQUIRED — Round [N] violation report:

  BASH RESULTS (정확한 수치):
  - TSC errors: [N]
  - Unguarded console: [N]
  - Explicit any: [N]
  - Circular deps: [N]
  - rgba violations: [N] (theme.ts 외)
  - hex violations: [N] (theme.ts 외)
  - Spacing literals: [N]
  - Secrets: [N]
  - AI in UI: [N]

  T1 LLM issues (있는 경우): [T1 breakdown]
  T4 LLM issues (있는 경우): [T4 failed list]

  Create FIX_PLAN: 파일별 구체적 수정 지시.
  Planner만 수정 방향을 결정. Sol이 Generator에게 전달.
  Mirror language."

Sol → FIX_PLAN을 plan file에 저장

Sol → Generator(Sonnet):
  "FIX_PLAN 실행:
  [FIX_PLAN 내용 — plan file에서 해당 섹션만]
  DO NOT make decisions. Execute exactly.
  List all modified files."

iteration += 1
GOTO Step 6A
```

**IF iteration > 15:**
```
=== HARD STOP: 15 iterations exceeded ===
Sol → AskUserQuestion:
  현재 상태: iteration=15, GM=[N]
  남은 violations: [목록]
  A) 수동 수정 후 재개
  B) 스코프 축소 후 재시작
  C) 현재 상태로 Ship Gate 진행 (GM<92 경고 포함)
```

### Step 6E: Breakthrough Mode (iteration >= 11)

```
Agent(model: "opus", prompt: "
  BREAKTHROUGH ANALYSIS — [N]회 루프 후 수렴 실패.

  ITERATION HISTORY: [각 라운드 T1-T5-GM 점수 + 위반 수치]
  CURRENT VIOLATIONS: [Bash 위반 목록]

  분석:
  1. 왜 수렴하지 않는가? (패턴 식별)
  2. 수정해도 다시 나타나는 위반이 있는가? (근본 원인)
  3. BUILD_PLAN 자체에 구조적 문제가 있는가?

  출력:
  - 근본 원인 (1-3개)
  - BUILD_PLAN 수정 제안 (필요시)
  - 즉시 적용 FIX (iteration 남은 경우)
  Mirror language.
")
```

Breakthrough 결과에 따라 Sol이 BUILD_PLAN 수정 후 Generator 재실행 가능.

### Step 6F: Ratchet (점수 역행 감지)

```
이전 라운드 GM보다 현재 GM이 낮으면:
  경고 출력: "⚠ RATCHET: GM [이전] → [현재] (-[차이])"
3라운드 연속 하락 → Breakthrough Mode 강제 진입 (iteration 11 이전이라도)
```

---

## PHASE 7: SHIP GATE

```
╔══════════════════════════════════════════════════╗
║         LUNA — SHIP READY                        ║
╠══════════════════════════════════════════════════╣
║  T1=[N] T2=[N] T3=[N] T4=[N] T5=[N] GM=[N.N]   ║
║  Consecutive PASS: 3/3                           ║
║  Iterations: [N]/15                              ║
║  Total cost: opus=[N] sonnet=[N] haiku=[N] calls ║
╚══════════════════════════════════════════════════╝
```

AskUserQuestion:
- A) Ship — git commit + 배포 단계 진행
- B) More polish — Phase 6 추가 라운드
- C) Pause — checkpoint 저장

```
<!-- CHECKPOINT: phase=7 status=shipped timestamp=ISO -->
```

---

## IMPORTANT RULES (Sol이 항상 따르는 규칙)

1. **Sol은 판단하지 않는다.** 데이터 전달 + 흐름 제어만.
2. **Generator는 결정하지 않는다.** Planner 지시 실행만.
3. **T2+T5 FAIL = T1/T4 LLM 호출 없음.** 기계 체크가 먼저.
4. **Bash 결과는 LLM 점수보다 우선.** tsc 에러 = T2 FAIL. 재량 없음.
5. **각 Agent에 plan file 전체를 넘기지 않는다.** 해당 Agent 필요 섹션만.
6. **FIX_PLAN에는 반드시 정확한 위반 수치가 포함된다.** Planner가 추측하지 않도록.
7. **3연속 PASS = SHIP.** 완벽한 점수보다 3회 일관성.
8. **Ratchet 3회 연속 하락 = Breakthrough 강제 진입.**
9. **Phase 5 Quality Rules는 Generator 프롬프트에 명시된다.** 레거시 복사 시도 포함.
10. **체크포인트는 각 Phase 완료 시 즉시 plan file에 기록.**
