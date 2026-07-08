# Claude Code Mastery Track

### From "prompt and accept" to building on the Agent SDK — mapped to your project ladder

**Design principle: just-in-time.** Every Claude Code feature is introduced at the exact milestone where the project makes it _necessary_, never as an isolated tutorial. And there's a meta-loop running through the whole track: **Claude Code is itself a production AI agent** — a harness with a model, tools, context management, and human-in-the-loop permissioning. From Project 3 onward, the mentor uses the tool you're holding as a living case study of the thing you're building. That dual perspective (user of an agent + builder of agents) is the single highest-value framing for your cohort.

---

## The Track

### Level 0 — Setup Ritual (one evening, before P1)

**Learn:** install & auth · the permission model and why it exists (this IS human-in-the-loop agent design — never reach for `--dangerously-skip-permissions` outside a sandbox) · **plan mode** (Shift+Tab — your default for anything non-trivial) · `/init` to generate CLAUDE.md, then replace it with the kit template · `/clear` vs `/compact` · Esc to interrupt · checkpoints & `/rewind` for fearless experimentation · `@file` references · extended thinking ("think hard" for architecture decisions)
**Habit installed:** _plan → review plan → approve → build → verify._ Never "prompt and accept" again.

### P1 (LLM Gateway) — Core Interactive Mastery

**Learn:** context hygiene as a discipline — `/clear` between milestones, watch for degradation, `/compact` at natural breakpoints (the same "chats are RAM" lesson, now in the terminal) · CLAUDE.md engineering: short, always-true conventions only — if it's a sometimes-workflow, it belongs elsewhere (you'll learn where in P2) · commit-as-you-go with descriptive messages · using checkpoints to try two implementations of the same SPEC and compare
**Why here:** P1 has 6 milestones — enough reps to make plan-mode + context hygiene automatic before anything fancier arrives.

### P2 (Production RAG) — Skills (part 1) + MCP as a Consumer

**Learn:** author your first **skills** in `.claude/skills/<name>/SKILL.md` — start with `build-notes` (generates your dissection fodder) and `dissect-prep` (packages diff + notes for Claude Chat). Learn the invocation model: explicit `/skill-name` or **auto-invoked when the description matches the task** — writing good descriptions is your first taste of tool-routing design · connect your first **MCP server as a consumer** (e.g., a Postgres MCP to inspect your pgvector tables from inside Claude Code)
**Why here:** your session loop has stabilized into repeatable workflows — exactly what skills exist to encode. And consuming MCP now sets up building MCP in P4.

### P3 (Agents from Scratch) — Subagents + Claude Code as Case Study

**Learn:** create **subagents** in `.claude/agents/` — a `code-reviewer` (restricted to read-only tools: Read, Grep, Glob) and a `test-runner` that runs the suite in its own context and reports back only failures. The lesson underneath: **context isolation** — verbose work stays in the subagent's window, only summaries return · pin cheaper models to grunt-work subagents (haiku for classification, sonnet for review) — this is cost-based model routing, the exact pattern you'll productionize in P8 · run subagents in parallel
**The meta-lesson:** as you hand-build your agent loop, the mentor dissects Claude Code alongside it — its tool schemas, its permission prompts as human-in-the-loop, its context strategies. You're reverse-engineering the best agent harness in production while writing your own.

### P4 (MCP + Multi-Agent) — Build MCP Servers + Parallel Sessions + Agent Teams

**Learn:** register YOUR OWN MCP servers (built in this project) into Claude Code via `.mcp.json` — your retriever and tools become native Claude Code capabilities · **git worktrees** for running parallel Claude Code sessions on isolated branches · **Agent Teams** — separate coordinated processes with bidirectional communication, versus subagents' one-way report-back
**Why here:** the project IS orchestrator–worker multi-agent; your tooling now mirrors your architecture.

### P5 (LLMOps: Evals & Guardrails) — Hooks + Headless + CI

**Learn:** **hooks** — deterministic lifecycle control: a `PreToolUse` hook that blocks reads of `.env` and secrets (exit code 2 = deny), a `PostToolUse` hook that auto-runs ruff/eslint + relevant tests after every edit · **headless mode** (`claude -p`) — one-shot, no TTY, same settings/hooks/permissions as interactive — wired into **GitHub Actions** so your prompt-regression eval gate runs on every PR · scheduled tasks for nightly eval runs
**The rhyme:** hooks are guardrails for your coding agent in the same week you're building guardrails for your product's agents. PreToolUse : Claude Code :: input validation : your gateway.

### P6 (Voice + Multimodal) — Multimodal Inputs + Browser MCP

**Learn (light week):** paste screenshots into the terminal for UI debugging · Playwright MCP server so Claude Code can drive a browser to test your voice-app frontend end-to-end
**Why light:** P6's project load is heavy; the CC additions ride along for free.

### P7 (Fine-Tuning) — Advanced Skill Authoring

**Learn:** multi-file skills with supporting scripts — a `data-curation` skill (dedupe/filter/format scripts + instructions) and a `gpu-cost-estimator` skill · skill authoring best practices: the description field is the router, keep SKILL.md lean, push detail into supporting files loaded on demand
**Why here:** fine-tuning workflows are procedural and repetitive — perfect skill material, and your cohort will reuse these exact skills.

### P8 (Inference Infra) — Plugins + Your Marketplace

**Learn:** bundle everything you've built — skills, subagents, hooks, MCP server definitions — into a **plugin** (`.claude-plugin/plugin.json` manifest) · publish to your own marketplace repo · namespacing (`your-plugin:skill-name`)
**The payoff:** "install my AI-Engineering plugin" becomes your cohort's day-one onboarding — one command and every student has your entire toolkit.

### Capstone — The Agent SDK (Python + TypeScript)

**Learn:** the **Claude Agent SDK** — Claude Code's engine as a library. `query()` + `ClaudeAgentOptions`, custom tools, programmatic hooks, subagents from code, MCP server wiring, permission modes, session management, loading your P8 plugin programmatically
**Full circle:** your capstone product (the AI support engineer) runs ON the SDK. You've gone from using the agent → dissecting the agent → extending the agent → embedding the agent in your own product. That's the complete arc, and it's the story your cohort content tells.

---

## Mentor Prompt Extension — "CLAUDE CODE COACH"

_Append this block to the Mentor System Prompt (template 1) in every claude.ai Project's instructions._

```
# CLAUDE CODE COACH
Beyond AI Engineering concepts, you coach me to mastery of Claude Code itself
(terminal CLI). I am currently basic-level: I prompt it and accept edits.

RULES:
1. JUST-IN-TIME, ONE AT A TIME. Follow the Claude Code Mastery Track in
   docs/CC-TRACK.md. Introduce at most ONE new Claude Code feature per
   session, only when the current milestone makes it useful. Teach it with
   the same Teaching Protocol (Mode A/B/C) as any other concept.
2. PRE-FLIGHT REVIEW. When you hand me a SPEC, also tell me HOW to run this
   in Claude Code: plan mode or not, what to @-reference, whether to /clear
   first, which skill/subagent to use if one exists. If I describe my
   intended approach, critique it before I go.
3. POST-FLIGHT DEBRIEF. When I return with results, ask one question about
   my Claude Code usage (e.g., "did you review the plan before approving?",
   "how much context did you burn — should that have been a subagent?").
   Coach the workflow, not just the code.
4. CLAUDE CODE AS CASE STUDY. From Project 3 onward, when teaching agent
   concepts (tool design, context isolation, permissioning, routing,
   multi-agent), explicitly map them to how Claude Code implements them.
   I am building what I am using — exploit that constantly.
5. LEDGER. Claude Code techniques get tracked in docs/CC-SKILLS.md with the
   same 🔴🟡🟢 statuses. A technique is 🟢 only after I've used it unprompted
   in a later session. Include CC-SKILLS changes in every handoff STATE.md.
6. ARTIFACT BIAS. Whenever a workflow repeats twice, propose turning it into
   a skill, subagent, or hook — and make ME author it (you review). By P8
   these become my plugin.
```

---

## CC-SKILLS.md Ledger Starter

_Lives in `docs/` of every repo, same as CONCEPTS.md._

```
# Claude Code Mastery Ledger

| Technique | Status | Evidence |
|---|---|---|
| Plan mode discipline | 🔴 | |
| CLAUDE.md engineering | 🔴 | |
| /clear per milestone · /compact | 🔴 | |
| Checkpoints & /rewind | 🔴 | |
| Extended thinking triggers | 🔴 | |
| Skill authoring (single-file) | 🔴 | |
| MCP as consumer | 🔴 | |
| Subagents + model pinning | 🔴 | |
| .mcp.json (own servers) | 🔴 | |
| Git worktrees / parallel sessions | 🔴 | |
| Agent Teams | 🔴 | |
| Hooks (PreToolUse / PostToolUse) | 🔴 | |
| Headless (claude -p) + CI | 🔴 | |
| Multi-file skills w/ scripts | 🔴 | |
| Plugin packaging + marketplace | 🔴 | |
| Agent SDK (Python + TS) | 🔴 | |

🟢 requires: used it unprompted in a later session.
```

---

## Setup additions (do once)

1. Save this file as `docs/CC-TRACK.md` in each project repo (Claude Chat references it; Claude Code can read it too).
2. Append the CLAUDE CODE COACH block to your Project instructions in claude.ai.
3. Add one line to STATE.md's template under "Ledger changes": `CC-SKILLS: [technique] 🔴→🟡 ...`
4. Level 0 tonight: install, run through the Setup Ritual list top to bottom in a throwaway repo before P1 Milestone 1.
