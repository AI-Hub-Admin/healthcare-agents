# Karpathy-Style Agent Eval Loop

**Date**: 2026-03-21
**Status**: Approved
**Replaces**: `2026-03-20-exam-architect-eval-loop-design.md` (6,771-line Python pipeline — over-engineered)

## Problem

We have 51 healthcare administration AI agents (system prompt .md files in `agents/`; 52 files total but `eval-exam-architect.md` is a meta-agent from the previous design and is excluded from eval). We need a way to point at one, find its weaknesses, improve it, and produce a standardized score. Personal tool, not production system.

## Inspiration

Karpathy's autoresearch (~630 lines total):
- `program.md` — tells Claude Code what to do in the loop
- `prepare.py` — frozen metric the agent can't touch
- `train.py` — the only file the agent can edit
- Git as ratchet — commit if better, `git reset HEAD~1` if worse
- ~12 experiments/hour, ~100 overnight, zero API cost

The pattern has been replicated by dozens of people (Ralph Wiggum Loop, Continuous Claude, Anthropic's own skill eval loop). The universal structure is three things: a frozen metric, a bash/session loop, and git as ratchet.

## Design

### Three Files

1. **`/eval` slash command** — the program. Drives the loop. Can read and edit agent .md files.
2. **`eval/rubric.md`** — the frozen metric. Scoring criteria Claude-as-judge uses. The loop can NEVER modify this file.
3. **`eval/results.tsv`** — append-only log of every iteration.

### The Loop (up to 5 iterations per invocation)

```
1. Read agents/{name}.md
2. Generate 25 questions covering the agent's key responsibilities
3. Answer all 25 as the agent (using the .md as system context)
4. Judge each answer against rubric.md (score all 3 criteria per question)
5. Compute weighted score → 0-100
6. Identify the 2-3 weakest areas
7. Edit agents/{name}.md to strengthen weak areas
   - Must not exceed 120% of original line count (deterministic check via wc -l)
   - Prefer appending specific guidance to existing sections over rewriting
8. Re-answer the SAME 25 questions with the edited agent
9. Re-judge → new score
10. New > old → git add agents/{name}.md && git commit -m "eval: {agent} {before}→{after} (+{delta})"
    New <= old → git restore agents/{name}.md
11. Append to eval/results.tsv
12. Next iteration (generate fresh 25 questions)
```

### Invocation

```
/eval revenue-medical-coding-specialist
```

One agent per session. Do not run multiple agents in a single invocation — context window fills after ~325K tokens (5 iterations × ~65K each), and judgment quality degrades with too much prior conversation in context.

To run all agents, invoke separately:
```
/eval revenue-medical-coding-specialist
/eval compliance-officer
/eval prior-authorization-specialist
...
```

### The Frozen Rubric

`eval/rubric.md` — immutable. The loop cannot touch this file.

**Scale**: 0-4 per criterion per question.

| Level | Score | Meaning |
|-------|-------|---------|
| Exemplary | 4 | Correct, comprehensive, cites specific regulations/codes by name, immediately actionable |
| Proficient | 3 | Correct, covers main points, references specific codes or sections, practical |
| Developing | 2 | Mostly correct but vague, missing key details, generic guidance |
| Novice | 1 | Partially correct, significant gaps, no specific citations |
| Incorrect | 0 | Factually wrong, hallucinated citations, or dangerously misleading |

**Criteria** (applied to every question):

| Criterion | Weight | What it measures |
|-----------|--------|-----------------|
| Accuracy | 40% | Factually correct, real citations, current guidelines. Score of 3+ requires specific codes or regulatory sections. Score of 4 requires named sources with publication dates or CFR numbers. |
| Completeness | 35% | All relevant steps covered, exceptions flagged, edge cases noted, nothing critical omitted |
| Specificity | 25% | Cites specific codes/sections/timelines, actionable recommendations, not generic advice |

**Score computation**: Each question yields 3 scores (0-4). Apply weights (0.40, 0.35, 0.25) for a weighted score per question (0-4). Average across 25 questions. Multiply by 25 for 0-100 scale.

### Results Log

`eval/results.tsv` — append-only.

```
iteration	agent	score_pre_edit	score_post_edit	delta	status	weak_areas	description
1	revenue-medical-coding-specialist	68	74	+6	improved	accuracy:E/M coding	Strengthened 2021 AMA E/M guidelines section
2	revenue-medical-coding-specialist	71	69	-2	reverted	completeness:modifier logic	Attempted modifier 25/59 expansion, no improvement
```

Column names use `pre_edit` / `post_edit` to make clear these are same-question-set comparisons within a single iteration. The score_pre_edit of iteration 2 (71) is NOT the score_post_edit of iteration 1 (74) — they use different question sets.

**Status values**: `improved` (committed), `reverted` (checked out), `capped` (edit exceeded 120% line count, reverted without re-scoring).

**Important**: Scores across iterations are NOT comparable. Each iteration uses different questions. Only the delta within an iteration (same questions, before vs after) is meaningful.

### Commit Gate

Karpathy-simple:
- Score improved → `git add agents/{name}.md && git commit -m "eval: {name} {before}→{after} (+{delta})"`
- Score same or worse → `git restore agents/{name}.md`

No threshold. No regression check beyond the same-question re-score. The ratchet self-corrects over iterations.

### Edit Guardrails

Two deterministic constraints on the edit step:

1. **Line count cap**: After editing, run `wc -l agents/{name}.md`. If it exceeds 120% of the pre-edit line count OR pre-edit + 50 lines (whichever is greater), revert immediately without re-scoring. Log as `status: capped`. The floor of 50 lines prevents the cap from blocking legitimate improvements on shorter files.
2. **Append-preferred**: The program instructs Claude to prefer adding specific guidance to existing sections over rewriting or reorganizing. Soft constraint but biases toward targeted fixes.

### Question Persistence

Questions are ephemeral — they exist only in the conversation context during the iteration. They are NOT written to disk. This is intentional:

- Writing questions to disk adds file I/O complexity for no benefit (questions are single-use within an iteration)
- The same-question comparison happens within one iteration, so both scoring passes see the questions in context
- Fresh questions each iteration prevents overfitting to a specific question set
- If debugging is needed, the results.tsv `weak_areas` and `description` columns capture what matters

By iteration 5, the context contains ~5 iterations of Q&A (~325K tokens). This is within the 1M context window but near the practical limit. The slash command should cap at 5 iterations and end the session.

### Git Branch Policy

Run `/eval` on a feature branch (e.g., `eval/{agent-name}`), not on main. The ratchet commits directly to the current branch. Merge to main when satisfied with results.

### Known Limitations (Accepted for v1)

1. **Claude judges Claude** — same model generates questions, answers them, and judges them. Systematic bias cancels out in same-question comparison (bias applies equally to before and after scores). Remaining risk: Claude rewards confident-sounding wrong answers. Mitigated by rubric requiring specific citations for high accuracy scores.

2. **Stochastic metric** — unlike Karpathy's deterministic BPB, our judge has variance. Same-question comparison adds structural determinism (the only variable is the agent .md edit). Across iterations, noise averages out.

3. **No gold standard** — v2 could add 5-10 hand-written question-answer pairs per agent as a canary set. Not needed for v1.

4. **No cross-agent comparison** — the standardized score (0-100) uses the same rubric across all agents, so scores are comparable in principle. But question difficulty varies by agent domain, so a 75 on medical coding is not the same difficulty as a 75 on patient experience.

## What This Replaces

The previous design (`2026-03-20-exam-architect-eval-loop-design.md`) specified:
- 10-stage Python pipeline (6,771 lines across 24 files)
- DSPy GEPA/MIPROv2 optimizer integration
- IRT 2PL/3PL psychometric calibration
- 3-judge panel with weighted kappa agreement
- Pydantic v2 schemas with 16 validation rules
- Anthropic API with prompt caching
- Estimated cost: ~$17,850 for 51 agents

This design specifies:
- 3 files (slash command, rubric, results log)
- Runs inside Claude Code (zero API cost)
- No Python, no dependencies
- Estimated time: ~15-20 minutes per agent per session

The previous infrastructure (eval/schema/, eval/harness/, agents/eval-exam-architect.md) can be archived or deleted. The rubric files in eval/rubrics/ informed the rubric.md design and can be referenced but are not used by this system.

## File Manifest

| File | Purpose | Mutable by loop? |
|------|---------|-------------------|
| `.claude/commands/eval.md` | The program — loop instructions | NO (read-only) |
| `eval/rubric.md` | Frozen scoring metric | NO (never) |
| `eval/results.tsv` | Iteration log (git-tracked) | APPEND-ONLY |
| `agents/{name}.md` | The thing being optimized | YES (the only mutable target) |
