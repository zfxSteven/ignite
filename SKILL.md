---
name: ignite
description: >
  Triggered by `/ignite <task/idea>`. Multi-agent plan review system: takes a one-line task or idea,
  guides the user to clarify requirements, generates a plan, then runs 3 expert Suggestors to critique,
  structured debates between Planner and Suggestors (~3 rounds), an impartial Judge to deliver verdicts,
  user confirmation, and a revised plan. Use this skill ONLY when the user types `/ignite` or explicitly
  asks to run the ignite plan review process.
---

# Ignite — Multi-Agent Plan Review

The user has invoked `/ignite` with a task or idea. Your job: take that raw input and run it through a structured multi-agent review pipeline that produces a debate-tested, user-confirmed plan.

The core insight: a single AI, no matter how capable, has blind spots. By forcing multiple expert perspectives to argue with each other and the planner, you surface assumptions, catch risks, and produce better plans than any single viewpoint could.

## Process Overview

```
Stage 0: Clarify Requirements    →  Task Document (structured)
Stage 1: Generate Plan           →  Plan v1
Stage 2: Multi-Perspective Critique →  9-15 Suggestions (3 Suggestors × 3-5 each)
Stage 3: Structured Debate       →  Debated Suggestions (up to 3 rounds each)
Stage 4: Judge's Verdict         →  ADOPT / MODIFY / REJECT per suggestion
Stage 5: User Confirmation       →  User decides final list
Stage 6: Revise Plan             →  Plan v2 (with diff markers)
Stage 7: Update Project State    →  Append to PROJECT_STATE.md
```

## File Persistence

On first use, create `./task_cache/` in the project root. Each task gets a subdirectory:

```
task_cache/
  task-<slug>-<date>/
    00_task.md           ← Stage 0
    01_plan_v1.md        ← Stage 1
    02_suggestions.md    ← Stage 2
    03_debates.md        ← Stage 3
    04_verdicts.md       ← Stage 4
    05_user_decisions.md ← Stage 5
    06_plan_v2.md        ← Stage 6
```

Write each file at the end of its stage. For iteration 2+, append version suffixes: `03_debates_v2.md`, `06_plan_v3.md`.

---

## Stage 0: Requirement Clarification

The user's input might be a one-liner. Your job: fill in the gaps through 2-3 rounds of focused questions, then synthesize into a Task Document.

**What to ask about** (pick 2-3 per round, don't dump all at once):
- **Goal**: Core problem? What does success look like?
- **Context**: Who are the users? Current state vs desired state?
- **Scope**: Hard constraints? (time, budget, team size, technology)
- **Stakeholders**: Who cares about the outcome? Who has decision power?

**When to stop asking**: You have enough when you can fill in all 6 sections of the Task Document below. Don't over-milk — if the user has already given you most of this, just confirm and move on.

**Task Document template:**

```markdown
# Task: [Clear title]

## Background
[1-2 paragraphs]

## Objectives
- [Primary]
- [Secondary]

## Scope
- In scope: [...]
- Out of scope: [...]

## Constraints
- [Time / budget / team / tech]

## Success Criteria
- [Measurable outcomes]

## Users / Stakeholders
- [Who uses it, who is affected]
```

Present to the user for confirmation before Stage 1.

---

## Stage 1: Plan Generation

Spawn a **Planner** subagent. Give it the Task Document. Ask for:
- Work broken into phases with deliverables
- Concrete steps (not vague milestones)
- Key decisions flagged
- Dependencies between steps
- Risks and mitigations

The plan should be detailed enough that a competent team could start executing it. Present a brief summary to the user, then move to Stage 2.

---

## Stage 2: Multi-Perspective Critique

Spawn **3 Suggestor subagents** in parallel. Each represents a different expert perspective.

### Choosing perspectives

Match the task domain. If unsure, propose a set to the user and explain why:

| Domain | Suggestor 1 | Suggestor 2 | Suggestor 3 |
|--------|-------------|-------------|-------------|
| Software development | Product Manager | Tech Lead | Operations / Business |
| Product launch | Marketing | Sales | Customer Success |
| Research project | Domain Expert | Methodologist | Stakeholder |
| Business strategy | Finance | Market Analyst | Operations |
| Team / Org change | HR / People | Team Lead | Individual Contributor |

Custom perspectives are fine — just explain to the user why these 3 cover the most ground.

### Suggestion quality matters

Each Suggestor produces **3-5 suggestions**. The key to good suggestions is **specificity**:

```
### Suggestion [N]: [Short title]

**Point**: [What specifically should change in the plan — reference the specific module/phase]
**Reasoning**: [Why this matters, with evidence or concrete reasoning — not just opinion]
**Severity**: [High / Medium / Low]
**Suggested action**: [A concrete, implementable change — not "consider doing X"]
```

**Good suggestion**: "M2 notification module uses synchronous webhook calls which will block the API under high load (e.g. during batch imports). Add Redis queue for async processing."

**Bad suggestion**: "The notification system could be improved for performance."

If a Suggestor can't find 3 meaningful suggestions, fewer is fine — it might mean the plan is solid from that angle.

---

## Stage 3: Structured Debate

This is the core value of the skill. **Every High-severity suggestion gets a debate.** For Medium/Low, group similar ones and debate together or skip if the Planner agrees.

### How to debate

For each suggestion, spawn a subagent that moderates the debate. The Planner defends the plan; the Suggestor argues for the change. Both sides must bring reasoning, not just opinions.

**Debate rules:**
- Up to 3 rounds (6 exchanges). Either side may concede early.
- Each response must address the specific point raised — no hand-waving or "noted, we'll consider it."
- The Planner should push back when they disagree — not just capitulate. Real debate produces real insights.
- The Suggestor should adjust their position if the Planner raises valid counterpoints. This isn't about winning.

**Output format** (use exactly):

```
**Round 1:**

【[Suggestor role] 提出建议】
[Argues for the suggestion — specific evidence, not just "this is important"]

【PLANNER 回应】
[Responds — agrees, pushes back with specific reasoning, or proposes middle ground]

**Round 2:**

【[Suggestor role] 进一步论证】
[Addresses the Planner's specific points — adjusts position if needed, strengthens argument]

【PLANNER 回应】
[Responds to new points — accepts if convinced, refutes with evidence if not]

**Round 3:**

【[Suggestor role] 最终表态】
[Final argument or concession]

【PLANNER 最终回应】
[Final response or acceptance]

---

**Debate Outcome:** [agreed / partially agreed / disagreed — and the key reasoning from each side]
```

Present debates grouped by Suggestor role. Run them in parallel where possible.

---

## Stage 4: Judge's Verdict

Spawn a **Judge** subagent — impartial, no stake in the plan. The Judge reads all suggestions and debate outcomes, then delivers verdicts.

**Judgment criteria:**
- Did the debate produce new insight that the original plan lacked?
- Is the suggested change feasible within the plan's constraints?
- Does it serve the task's stated objectives?
- If the specific proposal isn't feasible, does the underlying idea still have value that could be adapted?

**Verdict types:**
- **ADOPT** — The suggestion is clearly beneficial and feasible. Integrate as-is.
- **MODIFY** — The direction is right but the specific proposal needs adjustment. This is the most common verdict for ideas that are good in principle but need to be adapted to the team's actual resources and capacity. For example: a rigorous A/B test won't work for a 12-person team, but a before/after comparison preserves the same data-driven thinking at lower cost. State what to change and why the adaptation works better.
- **REJECT** — The specific execution doesn't work within current constraints, but the underlying thinking may still have value. A REJECT is not "this idea is wrong" — it's "not now, not this way." Every REJECT must include a preserved insight.

**Output format:**

```
### Suggestion [N]: [Title]
**Verdict: [ADOPT / MODIFY / REJECT]**
**Summary**: [1-2 sentence summary of both sides' key arguments]
**Reasoning**: [Why this verdict — what was most convincing]
**Modification** (if MODIFY): [What specifically to adjust]
**Preserved Insight** (if REJECT): [What part of the thinking is worth keeping and why. This is not optional — every REJECT must have one.]
```

**Preserved Insight** is the key difference between a productive REJECT and a dismissive one. It captures: what was the core thinking, why it's valuable in principle, and when/where it might become relevant (e.g., "A/B testing is sound data-driven thinking — use it when the team scales beyond 20 people" or "this approach works in a different context"). These preserved insights go into a "Deferred Ideas" section at the end of the verdicts, so they're not lost.

If unsure, lean toward MODIFY. REJECT should be rare — most ideas have a kernel of value that can be adapted to constraints. Ask: "Can this idea work if I simplify it?" before rejecting. Present all verdicts in a summary table to the user.

---

## Stage 5: User Confirmation

Present verdicts grouped by type:

```
### ADOPT (N)
1. [Title] — [1-line summary]
2. ...

### MODIFY (N)
3. [Title] — [1-line summary]
4. ...

### REJECT (N) — Deferred Ideas
5. [Title] — [why rejected] → Preserved: [the insight worth keeping]
```

For each, show the key reasoning (1-2 lines, not the full debate). Then ask:

> "You can confirm all as-is, override any verdict, or ask for more details on any suggestion. What would you like to do?"

Record the user's decisions in `05_user_decisions.md`.

---

## Stage 6: Plan Revision

Spawn the **Planner** to revise the plan based on confirmed suggestions.

**Revision rules:**
- Integrate changes naturally into the plan structure — don't just append.
- Mark every change: **[Updated]** for modifications, **[NEW]** for additions. Use ~~strikethrough~~ for removed items.
- If two confirmed suggestions conflict, note it and explain the resolution.
- Preserve the overall structure — the user should be able to compare v1 and v2 side by side.

Present the revised plan with a changes summary:

```
## Plan v2 — Changes from v1
- [What changed + why]
- ...

## Full Plan v2
[The complete revised plan with diff markers]
```

Ask if satisfied or wants another iteration.

---

## Stage 7: Update Project State

Update `PROJECT_STATE.md` **in the current project's root directory** (i.e. the working directory where `/ignite` was invoked). Each project maintains its own independent `PROJECT_STATE.md` — do NOT write to the skill directory under `~/.claude/skills/ignite/`. This happens in two phases:

### Phase A: Plan completed (after Stage 6)

After the plan is finalized and saved, immediately record the task:

1. **"最后更新" date** at the top
2. **任务记录 table** — append a new row:
   - 任务名称, 日期, 状态=`方案完成`, 裁决摘要(X ADOPT / Y MODIFY / Z REJECT), Plan 版本, 目录路径
3. **统计** — recalculate totals
4. **关键洞察** — add cross-task patterns if any emerged

### Phase B: Implementation completed (after user acts on the plan)

When the user comes back and reports that they've acted on the plan (or during a future `/ignite` session), update the same task row:

- Status: `方案完成` → `已落地`
- Add an **执行备注** column noting key outcomes, deviations, or lessons learned

This two-phase approach ensures:
- No information gap — completed reviews are recorded immediately
- Outcome tracking — what actually happened is captured later
- Accumulated wisdom — patterns across "what worked / didn't work" feed back into future reviews

**If `PROJECT_STATE.md` doesn't exist yet**, create it with this structure:

```markdown
# Ignite — 项目状态看板

> 最后更新: YYYY-MM-DD

## 项目概述
[Description of the skill and process]

## 任务记录
| # | 日期 | 任务名称 | 状态 | 裁决摘要 | Plan 版本 | 目录 | 执行备注 |
|---|------|---------|------|---------|----------|------|---------|
| N | date | name | 方案完成/已落地 | X ADOPT / Y MODIFY | v1→v2 | path | (落地后补充) |

## 统计
- 总任务数: N
- 总建议数: N
- 采纳率: N%
- 修改率: N%
- 拒绝率: N%
- 落地率: N%

## 关键洞察
### 跨任务共性发现
[Patterns observed across multiple tasks]

## 项目结构
[Directory tree]
```

---

## Orchestrator Guidelines

- **Move fast when you can.** If the user's input is already well-formed, Stage 0 should be quick (1 round of questions, or just confirm). If it's a vague one-liner, invest in clarification.
- **Show don't dump.** Brief summaries between stages. Full detail when the user asks.
- **The debate is the product.** This is where the skill delivers value that a single-pass AI response cannot. If the debates are shallow, the whole exercise is wasted. Push for substance.
- **Stay neutral.** You're the orchestrator, not a participant. Don't tilt debates toward either side.
- **Respect the user's time.** Not every suggestion needs a full 3-round debate. Skip obvious ones (Planner agrees immediately) and focus effort on the contentious ones.
- **Edge cases are fine.** If a Suggestor finds the plan solid, fewer suggestions are OK. If all 3 flag the same issue, highlight it — that's a strong signal.
