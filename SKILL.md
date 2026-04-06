---
name: plan-sol-v3
preamble-tier: 3
version: 2.0.0
description: |
  MANUAL TRIGGER ONLY: invoke only when user types /plan-sol-v3.
  Sol Harness 5-Tier를 전체 파이프라인 백본으로 통합한 최종 시스템.
  Sol(오케스트레이터) → Planner(Opus) → Generator(Sonnet) → Evaluator(5-Tier) 피드백 루프.
  Phase 3 T1 구조검증, Phase 5 빌드, Phase 6 통합검증. ALL >= 90, GM >= 92.
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
  - WebSearch
---

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions 2>/dev/null || true
touch ~/.gstack/sessions/"$PPID" 2>/dev/null || true
_PROACTIVE=$(~/.claude/skills/gstack/bin/gstack-config get proactive 2>/dev/null || echo "true")
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
source <(~/.claude/skills/gstack/bin/gstack-repo-mode 2>/dev/null) || true
REPO_MODE=${REPO_MODE:-unknown}
_TEL_START=$(date +%s)
_SESSION_ID="$$-$(date +%s)"
mkdir -p ~/.gstack/analytics 2>/dev/null || true
echo '{"skill":"plan-sol-v3","ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","repo":"'$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "unknown")'"}'  >> ~/.gstack/analytics/skill-usage.jsonl 2>/dev/null || true
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

# /plan-sol-v3 — Sol Harness Backbone Architecture v2.0

> "품질은 마지막에 검사하는 게 아니라, 처음부터 만드는 것이다."

## PIPELINE

```
Phase 0: SOCRATES → Phase 1: CEO(Opus×2) → Phase 2: DESIGN(Sonnet)
→ Phase 3: ENG(Opus×2) + T1구조검증 + Test Plan
→ Phase 4: PLAN GATE
→ Phase 5: BUILD(Unit별 Sonnet + Haiku 즉시검증)
→ Phase 6: 5-Tier 통합검증(GM >= 92)
→ Phase 7: SHIP GATE
```

## SCORING

| Tier | Name | Pass | Phase 5 검증 | Phase 6 검증 |
|------|------|------|-------------|-------------|
| T1 | Prompt Quality | >= 90 | — | ★ 전체 |
| T2 | Code Quality | >= 90 | ★ 부분 (hex, any, tsc) | ★ 전체 |
| T3 | Visual UX | >= 90 | — | ★ 전체 |
| T4 | Functional | >= 90 | — | ★ 전체 |
| T5 | Deploy Safety | >= 90 | — | ★ 전체 |

**ALL >= 90, GM >= 92. 예외 없음.**
Phase 5 즉시검증 = T2 부분검증 (hex/spacing/any/console). T3는 Phase 6에서만 평가.

## MODEL ROUTING

| Task | Model |
|------|-------|
| Phase 0 Socrates | user's model |
| Phase 1 CEO Primary + Challenger | opus × 2 |
| Phase 2 Design | sonnet |
| Phase 3 Eng Primary + Challenger | opus × 2 |
| Phase 3 T1 구조검증 | haiku |
| Phase 5 Build per Unit | sonnet |
| Phase 5 즉시검증 per Unit | haiku |
| Phase 6 T1, T2, T5 | haiku (병렬) |
| Phase 6 T3, T4B | sonnet |
| Phase 6 T4A | haiku |
| Phase 6 Fix Decision | opus |
| Breakthrough (11+) | opus |

**CRITICAL**: Subagents CANNOT spawn subagents. Main thread orchestrates ALL.

## GLOBAL RULES

### Checkpoint
`<!-- CHECKPOINT: phase=N status=complete timestamp=ISO -->`
Resume: plan file 체크포인트 읽기 → AskUserQuestion(Resume/Restart/Jump).

### Context Budget
2000 token rule: raw > 2000 tokens → summarize before passing to Agent.

### Language Mirroring
Mirror user's language. Agent prompts: "Mirror the user's language."

### Section Skip List
디스크에서 review 스킬 파일을 읽을 때 건너뛸 섹션:
Preamble, AskUserQuestion Format, Completeness Principle, Search Before Building,
Contributor Mode, Completion Status Protocol, Telemetry, Step 0: Detect base branch,
Prerequisite Skill Offer, Outside Voice.

### Ratchet System (점수 역행 방지)
Phase 6에서 이전 라운드보다 점수가 하락하면 경고.
3라운드 연속 하락 시 Breakthrough Mode 강제 진입 (iteration 11 이전이라도).

### 6 Auto-Decision Principles
1. Completeness 2. Boil lakes 3. Pragmatic 4. DRY 5. Explicit over clever 6. Bias toward action
CEO: P1+P2. Eng: P5+P3. Design: P5+P1.

---

## PHASE 0: SOCRATES — Problem Discovery

### Rules
1. NEVER give direct answers 2. NEVER propose solutions 3. Guide through questions ONLY 4. Mirror language

### Step 0A: Context Gathering (Silent)
Read CLAUDE.md, TODOS.md, git log -20, referenced files. Build understanding silently.

### Step 0B: Opening Assessment
Ask ONE question: "이 기능이 해결하려는 핵심 문제가 뭐라고 생각하세요?"

### Step 0C: Guided Discovery (3-7 rounds)

| Level | Type | Example |
|-------|------|---------|
| 1 | Clarifying | "그 결론에 도달한 근거가 뭔가요?" |
| 2 | Probing | "Y가 없으면 어떻게 되나요?" |
| 3 | Connecting | "이게 Z와 어떤 관계가 있나요?" |
| 4 | Counter | "만약 A가 아니라 B라면요?" |
| 5 | Hypothetical | "프로덕션에 나가면 어떤 문제가 생길까요?" |

**Adaptation:**
- Right direction → deepen with next level
- Wrong direction → ask question that exposes contradiction
- "모르겠어요" → break into smaller sub-questions
- Asks for answer → "한 단계만 더 생각해보죠."

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
- COLORS: [primary, secondary, axis1, axis2, ...]
- SPACING: [xxs, xs, sm, md, lg, xl]
- FONTS: [headline, body]
```

Checkpoint: `<!-- CHECKPOINT: phase=0 status=complete -->`

---

## PHASE 1: CEO REVIEW (Opus × 2)

### Step 1A: Load Review Skill
```
Read ~/.claude/skills/plan-ceo-review/SKILL.md
```
If file not found → use embedded fallback:

**Embedded CEO Fallback:**
- Challenge all premises (list each, evaluate valid/assumed/wrong)
- Map existing code leverage (what can be reused)
- Dream state: CURRENT → THIS PLAN → 12-MONTH IDEAL
- Implementation alternatives (2-3 approaches with effort/risk/pros/cons)
- Temporal interrogation (HOUR 1 vs HOUR 6+)
- Scope decisions with rationale
- Error & Rescue Registry
- Failure Modes Registry
- NOT in scope section

### Step 1B: Primary CEO Review (Opus)
```
Agent(model: "opus", prompt: "
  You are a CEO/founder reviewing a development plan.
  [Insert loaded CEO skill content, SKIP sections per skip list]
  [OR use embedded fallback above if skill file missing]

  PLAN FILE CONTENTS: [plan file]
  PROBLEM DEFINITION CARD: [from Phase 0]

  Apply SELECTIVE EXPANSION. 6 principles (CEO: P1+P2 dominate).
  Reference Problem Definition Card in every scope decision.

  Required outputs:
  - Premise challenge with specific premises evaluated
  - Existing code leverage map
  - Dream state diagram (CURRENT → THIS PLAN → 12-MONTH IDEAL)
  - Implementation alternatives table (2-3 approaches)
  - Scope decisions with rationale
  - Error & Rescue Registry
  - Failure Modes Registry
  - NOT in scope section

  For each decision: | # | Decision | Principle | Rationale | Rejected |
  Mirror the user's language.
")
```

### Step 1C: Independent Challenger (Opus)
```
Agent(model: "opus", prompt: "
  Independent CEO/strategist. NO prior review.
  PLAN: [content]
  Evaluate: 1) right problem? 2) premises? 3) 6-month regret? 4) alternatives? 5) competitive risk?
  For each: what's wrong, severity (critical/high/medium), fix.
  Be adversarial. No compliments. Mirror language.
")
```

### Step 1D: CEO Consensus
```
CEO DUAL VOICES — CONSENSUS TABLE:
═════════════════════════════════════════════════
  Dimension                    Primary  Challenger  Consensus
  ─────────────────────────── ──────── ────────── ─────────
  1. Premises valid?            —        —          —
  2. Right problem?             —        —          —
  3. Scope correct?             —        —          —
  4. Alternatives explored?     —        —          —
  5. Competitive risks?         —        —          —
  6. 6-month trajectory?        —        —          —
═════════════════════════════════════════════════
```
**Premise Gate**: AskUserQuestion — premises require human judgment.

### Step 1E: Phase Transition
Write outputs to plan file. Emit:
```
=== PHASE 1 COMPLETE: CEO Review ===
Mode: SELECTIVE EXPANSION | Voices: [dual/single] | Consensus: [X/6]
<!-- CHECKPOINT: phase=1 status=complete -->
<!-- COST: phase=1 model=opus calls=2 -->
```

---

## PHASE 2: DESIGN REVIEW (Sonnet, conditional)

### Skip Condition
Grep plan for: component, screen, form, button, modal, layout, dashboard, sidebar, nav, dialog.
2+ matches → run. <2 → skip. Borderline (exactly 2) → AskUserQuestion.

### Step 2A: Dispatch
```
Read ~/.claude/skills/plan-design-review/SKILL.md (fallback: rate 7 dimensions below)
```

```
Agent(model: "sonnet", prompt: "
  Senior product designer reviewing plan.
  [Insert design skill or use embedded:]

  Rate 7 dimensions 0-10:
  1. Hierarchy (visual priority)
  2. Specificity (concrete decisions, not vibes)
  3. Empty states (designed, not afterthought)
  4. Interaction patterns (consistency)
  5. Accessibility (keyboard, contrast, touch 44pt)
  6. Visual coherence (brand system)
  7. Performance (perceived speed)

  PLAN: [content]
  CEO SUMMARY (max 500 tokens): [from Phase 1]
  6 principles (Design: P5+P1 dominate).
  Auto-decide structural. Defer taste. Mirror language.
")
```

### Step 2B: Checkpoint
```
=== PHASE 2 COMPLETE: Design Review ===
Score: [N]/10 | Taste decisions: [N]
<!-- CHECKPOINT: phase=2 status=complete -->
<!-- COST: phase=2 model=sonnet calls=1 -->
```

---

## PHASE 3: ENG REVIEW (Opus × 2) + T1 구조검증 + Test Plan

### Step 3A: Load + Dispatch Eng Review (Opus)
```
Read ~/.claude/skills/plan-eng-review/SKILL.md
```
If file not found → embedded fallback:

**Embedded Eng Fallback:**
- Architecture ASCII diagram (components + relationships)
- Test diagram (codepaths → coverage)
- Code quality review (DRY, error handling, naming)
- Performance analysis (N+1, caching, bottlenecks)
- Security review (auth, injection, PII)
- Deployment plan (reversibility, feature flags)

```
Agent(model: "opus", prompt: "
  Senior engineering manager reviewing plan.
  [Insert eng skill or use embedded fallback]

  PLAN: [content]
  CEO SUMMARY (max 500t): [from Phase 1]
  DESIGN SUMMARY (max 300t): [from Phase 2 or 'skipped']

  Required outputs:
  1. Architecture ASCII diagram
  2. Test diagram (codepaths → coverage)
  3. Test plan artifact (WRITE TO DISK)
  4. Failure modes registry
  5. NOT in scope
  6. What already exists

  NEVER skip Test Review. NEVER reduce scope (P2).
  Eng: P5+P3 dominate. Mirror language.
")
```

### Step 3B: Independent Eng Challenger (Opus)
```
Agent(model: "opus", prompt: "
  Independent senior engineer. NO prior review.
  PLAN: [content]
  Evaluate: 1) Architecture? 2) Edge cases 10x? 3) Tests missing? 4) Security? 5) Hidden complexity?
  Be adversarial. Mirror language.
")
```

### Step 3C: Eng Consensus Table
```
ENG DUAL VOICES — CONSENSUS TABLE:
═════════════════════════════════════════════════
  1. Architecture sound?      2. Test coverage?
  3. Performance risks?       4. Security threats?
  5. Error paths?             6. Deployment risk?
═════════════════════════════════════════════════
```

### Step 3D: Write Test Plan Artifact
Eng Review의 핵심 산출물. 반드시 디스크에 작성:
```bash
SLUG=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)" 2>/dev/null || echo "project")
mkdir -p ~/.gstack/projects/$SLUG 2>/dev/null || true
# Write test plan to ~/.gstack/projects/$SLUG/test-plan-$(date +%Y%m%d).md
```

### Step 3E: ★ T1 구조검증 (프롬프트 설계만 — 콘텐츠는 Phase 6)

Plan file의 "Prompt Design" 섹션을 검증. 실제 코드가 아닌 **설계 구조**만 확인:

```
Agent(model: "haiku", prompt: "
  T1 STRUCTURE CHECK — Plan file의 'Prompt Design' 섹션 검증

  확인:
  1. 페르소나 이름/성격 정의가 있는가? (PASS/FAIL)
  2. 축별 톤 분기가 명시되어 있는가? (PASS/FAIL)
  3. 지원 언어가 나열되어 있는가? (PASS/FAIL)
  4. system/user 역할 분리가 설계되어 있는가? (PASS/FAIL)
  5. AI 용어 금지 규칙이 있는가? (PASS/FAIL)
  6. 행동 제안 의무가 있는가? (PASS/FAIL)
  7. Design Tokens 섹션에 COLORS/SPACING/FONTS가 명시되어 있는가? (PASS/FAIL)

  7개 중 6개 이상 PASS → T1_STRUCT: PASS
  5개 이하 → T1_STRUCT: FAIL (누락 항목 리스트)
")
```

T1_STRUCT FAIL → 메인 스레드가 plan file 보완 후 재검증 (max 2회).
**Phase 4 진입 조건: T1_STRUCT PASS.**

NOTE: 이것은 "설계가 있는가"만 확인. "품질이 90점 이상인가"는 Phase 6 T1에서만 평가.

### Step 3F: Checkpoint
```
=== PHASE 3 COMPLETE: Eng Review ===
Architecture: [summary] | Test plan: [path] | Consensus: [X/6]
T1 Structure: [PASS/FAIL]
<!-- CHECKPOINT: phase=3 status=complete -->
<!-- COST: phase=3 model=opus calls=2 model=haiku calls=1 -->
```

---

## PHASE 4: PLAN GATE

### Pre-Gate Verification
Check all outputs exist in plan file:
- [ ] Problem Definition Card (Phase 0)
- [ ] Prompt Design section with all 7 fields (Phase 0E)
- [ ] Design Tokens section (Phase 0E)
- [ ] CEO review + consensus table (Phase 1)
- [ ] Design review outputs (Phase 2, if applicable)
- [ ] Eng review + consensus table (Phase 3)
- [ ] Test plan artifact on disk (Phase 3D)
- [ ] T1 Structure PASS (Phase 3E)
- [ ] Decision audit trail

Missing → produce (max 2 retries).

### Summary
```
╔══════════════════════════════════════════════════╗
║        PLAN SOL V3 — REVIEW COMPLETE             ║
╠══════════════════════════════════════════════════╣
║  Phase 0: Problem [one-line] | Rounds [N]        ║
║  Phase 1: CEO [mode] | Consensus [X/6]           ║
║  Phase 2: Design [N]/10 (or SKIPPED)             ║
║  Phase 3: Eng [sound/concerns] | Test plan [path]║
║  T1 Structure: PASS                              ║
║  Decisions: auto [N] / taste [N]                 ║
╚══════════════════════════════════════════════════╝
```

### AskUserQuestion: Plan Gate
- A) Approve → Phase 5
- B) Approve with overrides → specify taste decisions
- C) Interrogate → ask questions
- D) Revise → re-run affected phases (max 3 cycles)
- E) Reject → Phase 0
- F) Pause → save checkpoint

`<!-- CHECKPOINT: phase=4 status=approved -->`

---

## PHASE 5: BUILD — Sol이 Planner 지시를 Generator에게 전달

### 역할 분리

```
Sol (오케스트레이터 = 메인 스레드):
  - 판단하지 않음. 데이터를 전달하고 흐름을 제어.
  - Planner 결과 → plan file 반영 → Generator에게 전달
  - Generator 결과 → Evaluator에게 전달
  - Evaluator 결과 → Planner에게 전달 (FAIL 시)

Planner (Opus): "뭘 어떻게 만들지" + "뭘 어떻게 고칠지"
Generator (Sonnet): Planner 지시대로 코드 작성 (판단 안 함)
Evaluator (5-Tier): 결과물 검증 + 점수 (수정 안 함)
```

### Step 5A: Info Gathering (Haiku)
```
Agent(model: "haiku", prompt: "
  Read all files referenced in eng review architecture + existing code map.
  Output CONTEXT BRIEF (max 3000 tokens): exports, patterns, integration points.
")
```

### Step 5B: Planner — 빌드 계획 수립 (Opus)

Sol이 Planner에게 빌드 계획을 요청:

```
Agent(model: "opus", prompt: "
  You are the Planner. Create a BUILD PLAN for the Generator.

  PLAN FILE: [plan file content]
  CONTEXT BRIEF: [from 5A]
  DESIGN TOKENS: [from plan file — COLORS, SPACING, FONTS]
  TEST PLAN: [from Phase 3D]

  Divide work into Units (의존성 순서):
  Unit 1: 설정 (theme.ts, types, config) → 기반
  Unit 2: 유틸리티 (계산 엔진) → Unit 1
  Unit 3: 서비스 (GPT, auth, DB) → Unit 1-2
  Unit 4: 스토어 (state) → Unit 1-3
  Unit 5: 컴포넌트 — 신규 → Unit 1
  Unit 6: 컴포넌트 — 레거시 복사+토큰화 → Unit 1
  Unit 7: 스크린 → Unit 1-6
  Unit 8: 네비게이션 + App.tsx → Unit 1-7
  Unit 9: DB + Edge Functions + 웹 → Unit 1

  For each Unit, specify:
  - 파일 목록
  - Quality Rules (COLORS 토큰만, SPACING 토큰만, FONTS 필수, any 금지 등)
  - 레거시 복사 시: 원본 경로 + 토큰화 매핑 (hex → COLORS.xxx)

  Output: BUILD_PLAN (Generator가 바로 실행할 수 있는 구체적 지시)
")
```

Sol이 BUILD_PLAN을 plan file에 저장.

### Step 5C: Generator — Planner 지시대로 빌드 (Sonnet)

Sol이 BUILD_PLAN을 Generator에게 전달. Unit별 순차 또는 병렬:

```
FOR each Unit (1→9, 의존성 순서):

  Agent(model: "sonnet", prompt: "
    You are the Generator. Execute the Planner's instructions exactly.

    BUILD_PLAN Unit [N]: [Planner가 작성한 구체적 지시]
    CONTEXT BRIEF: [from 5A]
    PREVIOUS UNITS: [이전 Unit 결과 요약]

    DO NOT make design decisions. Follow the Planner's plan exactly.
    - COLORS: Planner가 지정한 토큰만 사용
    - SPACING: Planner가 지정한 토큰만 사용
    - FONTS: Planner가 지정한 대로 적용
    - 레거시 복사: Planner가 지정한 토큰 매핑대로 변환

    After: list ALL files created/modified.
    Mirror language.
  ")
```

Sol이 Generator 결과를 수집. **검증은 여기서 하지 않음** — Phase 6으로 넘긴다.

### Step 5D: Checkpoint
```
=== PHASE 5 COMPLETE ===
Units: 9/9 built
<!-- CHECKPOINT: phase=5 status=complete -->
<!-- COST: phase=5 model=opus calls=1 model=sonnet calls=[N] model=haiku calls=1 -->
```

---

## PHASE 6: Evaluator → Sol → Planner → Generator 루프

### Step 6A: T1 + T2 + T5 병렬 (Haiku × 3)

**T1 — PROMPT QUALITY:**
```
Agent(model: "haiku", prompt: "
  T1 — PROMPT QUALITY (실제 코드 콘텐츠 검증)

  Read the project's GPT prompt/service files (openai.ts or equivalent).
  9 checks, each scored:

  Binary checks (100 base, subtract per violation):
  1. No 'AI'/'artificial' in user-facing OUTPUT templates? (-20 each occurrence)
  2. No markdown **bold**/## in output format instructions? (-10 each)
  4. Domain data fields (saju, card names, etc.) in prompts? (-15 if missing)
  5. System/user role separation in API calls? (-20 if mixed)
  6. 4 languages (ko/en/ja/zh) in prompt system? (-10 per missing language)
  8. Domain-specific data in interpretation prompts? (-10 if missing)
  9. Axis/mode-specific tone differentiation? (-10 if absent)

  Judgment checks (score 0-100 independently):
  3. Persona consistency (name, tone, character across languages)
  7. Advice actionability (concrete, specific, not vague)

  T1_SCORE = min(100 + sum(binary deductions), average(check3, check7))
  Clamped 0-100.

  Output: T1_SCORE: [N] with per-check breakdown.
")
```

**T2 — CODE QUALITY:**
```
Agent(model: "haiku", prompt: "
  T2 — CODE QUALITY

  5 sub-checks, each scored 0-100:
  1. TypeScript: `npx tsc --noEmit` → 0 errors = 100, -5 per error, min 0
  2. Lint: unguarded console NOT in `if (__DEV__)` → 100 - 10*count, min 70
  3. Type Coverage: explicit `: any` annotations (not in comments) → 100 - 3*count, min 0
  4. Circular Dependencies: import cycles → 100 - 15*count, min 0
     NOTE: lazy require() is NOT a cycle
  5. Security: hardcoded API keys, PII in client GPT prompts → 100 - 20*count, min 0

  T2_SCORE = average of 5 sub-scores. Clamped 0-100.
  Output: T2_SCORE: [N], sub-scores: [ts/lint/type/circ/sec]
")
```

**T5 — DEPLOY SAFETY:**
```
Agent(model: "haiku", prompt: "
  T5 — DEPLOY SAFETY

  SCORING RULES (v2 교훈 반영):
  - 'AI' word: ONLY user-facing UI strings. SafeAreaView, variable names = NOT violations.
  - Console: ONLY outside `if (__DEV__)`. Guarded = OK.
  - Type `: any`: ONLY explicit type annotations. Comments/strings = NOT counted.
  - PII: Only CLIENT-SIDE raw birth_date/time/region in GPT prompts = violation.
    Server Edge Functions logging = NOT a client violation.
  - Unavailable checks: SKIP (0.25x weight), NOT FAIL.
  - lazy require() = NOT a circular dependency.

  12 checks:
  1. [critical] tsc --noEmit pass
  2. [warn] Test files exist
  3. [critical] No AI in user-facing strings
  4. [critical] No forbidden phrases ('I cannot', 'language model')
  5. [warn] All Edge Functions have index.ts
  6. [warn] .env.example complete
  7. [warn] Unguarded console <= 20
  8. [warn] src/ < 50MB
  9. [warn] explicit `: any` < 15
  10. [critical] No circular dependencies
  11. [critical] No secrets in source
  12. [critical] No PII in client GPT prompts

  Score = (pass + warn_pass*0.5 + skip*0.25) / 12 * 100
  Any critical FAIL = cap at 50.
  T5_SCORE: [N]
")
```

### Step 6B: T4 Functional (Haiku + Sonnet)

**T4A — Functional Tests (85%):**
```
Agent(model: "haiku", prompt: "
  T4A — FUNCTIONAL

  Verify against Test Plan artifact (from Phase 3D):
  1. Calculation functions produce correct results (test with known inputs)
  2. All screens export valid React components
  3. All stores have expected CRUD methods
  4. Edge Functions parse input correctly
  5. API response types match client expectations

  Each check: PASS (100) or FAIL (0). Average all.
  T4A_SCORE: [N]
")
```

**T4B — Boundary Mismatch (15%):**
```
Agent(model: "sonnet", prompt: "
  T4B — BOUNDARY MISMATCH

  8 checks, each 12.5 points (PASS=12.5, FAIL=0):
  1. All type union members used by at least one store/screen
  2. All cost/pricing entries cover all type members
  3. All navigation routes importable
  4. Theme tokens imported by screens (not hardcoded)
  5. Axis field consistent ('axis1'|'axis2') across types/stores/screens
  6. i18n keys in all supported languages
  7. Edge Function response shapes match client types
  8. Store hooks consumed by >= 50% of screens

  T4B_SCORE: [N] (0-100)
  T4_SCORE = T4A * 0.85 + T4B * 0.15
")
```

### Step 6C: T3 Visual UX (Sonnet)

```
Agent(model: "sonnet", prompt: "
  T3 — VISUAL UX (4 Layers)

  LAYER A — Binary Checks (35%, max 28pts):
  Each check has a max score. Sum all, divide by 28, multiply by 100.

  | # | Check | Max | Scoring |
  |---|-------|-----|---------|
  | 1 | Theme imported by main screens | 3 | +3 if yes, 0 if no |
  | 2 | No hardcoded hex/rgba | 5 | +5 if clean, -2 per violating FILE, min 0 |
  | 3 | SafeAreaView on all screens | 2 | +2 if all, 0 if any missing |
  | 4 | SPACING tokens used | 3 | +3 if clean, -1 per violating FILE, min 0 |
  | 5 | No text walls (>100char unbroken) | 2 | +2 if clean, -1 per wall, min 0 |
  | 6 | Touch targets >= 44pt | 3 | +3 if all, -2 per violating element, min 0 |
  | 7 | FONTS.headline/body applied | 2 | +2 if all screens, -1 per screen missing, min 0 |
  | 8 | Loading states present | 3 | +3 if key flows have loaders, 0 if none |
  | 9 | Error states present | 2 | +2 if key flows have error UI, 0 if none |
  | 10 | Empty states designed | 3 | +3 if first-time/no-data handled, 0 if none |

  EXEMPT from check 2: theme.ts definitions, SVG art files (major/*.tsx, TarotCardBack).
  EXEMPT from check 4: borderWidth, structural padding:1.

  LayerA = (sum of scores) / 28 * 100. Clamped 0-100.

  LAYER B — Design Dimensions (30%, 7 dims × 0-10):
  1. Hierarchy 2. Specificity 3. Empty states 4. Interaction
  5. Accessibility 6. Visual coherence 7. Performance UX
  LayerB = average * 10

  LAYER C — Independent UX Review (10%):
  As a fresh designer: identify top 3 real-user problems.
  Score: 100 - (critical × 20) - (medium × 10). Min 0.

  LAYER D — Code Consistency (25%, 4 checks × 25pts):
  1. Axis colors visually distinct in code (+25/0)
  2. Key animations use Reanimated (not plain Animated) (+25/0)
  3. Primary loader component implemented and IMPORTED by screens (+25/0)
  4. Design token sizes (CARD_SIZES etc.) consumed somewhere (+25/0)

  T3_SCORE = A*0.35 + B*0.30 + C*0.10 + D*0.25
  Output: T3_SCORE: [N], A=[N] B=[N] C=[N] D=[N]
")
```

### Step 6D: Scoring (Main Thread)

```
GM = exp(0.15*ln(T1/100) + 0.20*ln(T2/100) + 0.25*ln(T3/100) + 0.25*ln(T4/100) + 0.15*ln(T5/100)) * 100

VERDICT:
- ALL tiers >= 90 AND GM >= 92 → PASS
- Otherwise → FAIL

Scoreboard → write to plan file.
EMA = 0.5 * current_GM + 0.5 * previous_EMA
```

### Step 6E: FAIL 시 — Sol 오케스트레이션 PGE 루프

FAIL이면 Sol이 다음을 순차 실행:

**1. Sol → Planner (Opus): "뭘 고칠지 계획해"**
```
Agent(model: "opus", prompt: "
  HARNESS FAILED — Round [N], GM: [score]/92
  T1=[N] T2=[N] T3=[N] T4=[N] T5=[N]
  Failures: [per-tier details]

  You are the Planner. Analyze failures and create a FIX_PLAN.

  Prioritize by: (tier_weight × (90 - tier_score)) = GM impact
  For each fix: exact file path, exact change, expected score gain.
  
  Identify root cause:
  - Implementation flaw → specify which files to change
  - Plan flaw → recommend Phase 3 partial re-run
  - Evaluator too strict → recommend scoring rule adjustment

  Output: FIX_PLAN with ranked concrete actions.
  Mirror language.
")
```

**2. Sol: Planner의 FIX_PLAN을 plan file에 반영**
Sol이 FIX_PLAN을 plan file의 `## Fix Round [N]` 섹션에 기록.

**3. Sol → Generator (Sonnet): "Planner 지시대로 수정해"**
```
Agent(model: "sonnet", prompt: "
  You are the Generator. Execute the Planner's FIX_PLAN exactly.

  FIX_PLAN:
  [Sol이 전달하는 Planner의 구체적 수정 지시]

  DO NOT make your own decisions. Follow the Planner's instructions.
  After: list ALL files modified.
  Mirror language.
")
```

**4. Sol → Evaluator: Phase 6 재실행 (Step 6A~6D)**

이것이 PGE 루프 한 바퀴:
```
Evaluator(5-Tier) → Sol → Planner(Opus) → Sol → Generator(Sonnet) → Sol → Evaluator(5-Tier)
```

Sol은 데이터를 전달할 뿐, 판단하지 않음.

---

## VERIFICATION LOOP

```
iteration = 1, consecutive_pass = 0, previous_gm = 0

WHILE iteration <= 15:
  Run Phase 6 (5-Tier)
  current_gm = GM score

  IF ALL >= 90 AND GM >= 92:
    consecutive_pass++
    IF consecutive_pass >= 3: BREAK → Phase 7

  ELSE:
    consecutive_pass = 0

    // Ratchet check: 3연속 하락 시 강제 Breakthrough
    IF current_gm < previous_gm for 3 consecutive rounds:
      ENTER BREAKTHROUGH MODE (regardless of iteration count)

    // Standard Breakthrough at iteration 11
    IF iteration >= 11:
      ENTER BREAKTHROUGH MODE

    // EMA stagnation: 3 rounds delta < 0.5
    ema_delta = abs(current_ema - previous_ema)
    IF ema_delta < 0.5 for 3 consecutive rounds:
      ENTER BREAKTHROUGH MODE

  previous_gm = current_gm
  iteration++

IF iteration > 15:
  AskUserQuestion: "15 iterations reached. GM: [N]/92."
  - A) Continue manually
  - B) Restart Phase 5 with revised plan
  - C) Abort
```

### BREAKTHROUGH MODE
```
Agent(model: "opus", prompt: "
  META-ANALYSIS: [N] iterations, GM stagnant/declining.
  ALL ROUND SCORES: [history]
  FAILURE PATTERNS: [recurring issues]

  Root causes:
  1. Plan flaw (architecture doesn't support requirements)
  2. Approach wrong (implementation strategy fails)
  3. Evaluator too strict (false-fail patterns)
  4. Circular fix (A breaks B, B breaks A)

  Output: ROOT CAUSE [1-4] + CONCRETE ACTION
  If 3 (evaluator): recommend specific scoring rule adjustments.
  Mirror language.
")
```

---

## PHASE 7: SHIP GATE

### Summary
```
╔══════════════════════════════════════════════════╗
║           PLAN SOL V3 — COMPLETE                 ║
╠══════════════════════════════════════════════════╣
║  Socratic: [N] rounds                            ║
║  CEO: [mode] | [X/6] consensus                   ║
║  Design: [N]/10 (or SKIPPED)                     ║
║  Eng: [summary] | Test plan: [path]              ║
║  Build: [N] units | Verify: [N] rounds | GM [N]  ║
║  Convergence: [yes/no] | Breakthrough: [used/no] ║
║  Cost: Opus [N] | Sonnet [N] | Haiku [N]         ║
╚══════════════════════════════════════════════════╝
```

### AskUserQuestion
- A) Ship — commit
- B) Ship + changes — specify
- C) Re-verify — Phase 6 again
- D) Revise plan — Phase 4
- E) Reject — discard

`<!-- CHECKPOINT: phase=7 status=[shipped/rejected] -->`

---

## IMPORTANT RULES

1. **Never skip Phase 0** — Socratic discovery is the differentiator.
2. **Never abort** — respect the full pipeline.
3. **Problem Definition Card is the anchor**.
4. **Sequential** — 0→1→2→3→4→5→6→7.
5. **Log every decision** — no silent auto-decisions.
6. **Language mirroring**.
7. **Subagents cannot spawn subagents**.
8. **gstack graceful degradation**.
9. **Context budget** — 2000 token rule.
10. **Checkpoint after every phase** + COST tracking.
11. **3 consecutive PASS (GM >= 92) = ship**.
12. **15 iteration hard max**.
13. **Breakthrough at 11, OR 3 consecutive decline, OR EMA stagnation**.
14. **ALL tiers >= 90, GM >= 92** — 예외 없음.
15. **Phase 5 = Planner 지시 → Generator 실행** (검증은 Phase 6만). Quality Rules는 Planner가 빌드 계획에 포함.
16. **레거시 복사 = Planner가 토큰 매핑을 지시 → Generator가 변환 실행**. 있는 그대로 복사 금지.
17. **T1 구조검증** — Phase 3에서 plan file의 프롬프트 설계 구조만 확인 (PASS/FAIL). Phase 6 T1이 실제 품질 점수.
18. **Model per tier** — Haiku(T1,T2,T4A,T5,즉시검증), Sonnet(T3,T4B,Build), Opus(Fix,CEO,Eng).
19. **T5 채점 명확화** — AI=UI문자만, console=__DEV__가드 외만, any=타입선언만, PII=클라이언트만, lazy require≠cycle, unavailable=skip.
20. **Sol = 오케스트레이터** — 판단하지 않음. Planner 결과를 plan file에 반영 → Generator에게 전달 → Evaluator 결과를 Planner에게 전달.
21. **Ratchet** — 3연속 점수 하락 시 강제 Breakthrough.
22. **Phase transition emit** — 각 Phase 끝에 `=== PHASE N COMPLETE ===` + COST 기록.
23. **Test Plan Artifact** — Phase 3D에서 반드시 디스크에 작성.
24. **Plan file에 Design Tokens 섹션 필수** — Phase 5 Sonnet이 참조할 COLORS/SPACING/FONTS 매핑.
