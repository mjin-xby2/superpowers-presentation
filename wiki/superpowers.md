# Superpowers

Superpowers is a Claude Code plugin by Jesse Vincent (obra) that packages a full software-development methodology — brainstorming, planning, TDD, subagent-driven execution, and code review — into a chain of composable "skills" that auto-trigger during a coding session.

## 1. Skills in v6.3.0 (installed locally)

Version confirmed: `6.3.0` (per `package.json` and `RELEASE-NOTES.md`, dated 2026-08-12), installed at `/Users/mohammedhussain/.claude/plugins/cache/claude-plugins-official/superpowers/6.3.0/`.

14 skills, one per directory under `skills/`:

| Skill | Purpose (from SKILL.md frontmatter) |
|---|---|
| **brainstorming** | Must be used before any creative work — creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements, and design before implementation. |
| **using-superpowers** | Used at the start of any conversation — establishes how to find and use skills, requiring skill invocation before ANY response, including clarifying questions. |
| **using-git-worktrees** | Used when starting feature work needing isolation, or before executing plans — ensures an isolated workspace via native tools or a git worktree fallback. |
| **writing-plans** | Used once a spec/requirements exist for a multi-step task, before touching code. Breaks work into bite-sized (2–5 minute) tasks with exact file paths and verification steps. |
| **executing-plans** | Used when a written implementation plan exists, to execute it in a separate session with human review checkpoints (batch execution). |
| **subagent-driven-development** | Used to execute implementation plans with independent tasks in the current session — dispatches a fresh subagent per task with two-stage review. |
| **dispatching-parallel-agents** | Used for 2+ independent tasks that can proceed without shared state or sequential dependencies. |
| **test-driven-development** | Used when implementing any feature or bugfix, before writing implementation code — enforces RED-GREEN-REFACTOR. |
| **systematic-debugging** | Used on any bug, test failure, or unexpected behavior, before proposing fixes — a 4-phase root-cause process. |
| **verification-before-completion** | Used before claiming work is complete/fixed/passing, and before committing or opening PRs — requires running verification commands and confirming output; evidence before assertions. |
| **requesting-code-review** | Used when completing tasks or major features, or before merging, to verify work meets requirements. |
| **receiving-code-review** | Used when receiving code review feedback, before implementing suggestions — requires verification rather than reflexive agreement. |
| **finishing-a-development-branch** | Used once implementation is complete and tests pass — decides how to integrate the work (merge / PR / keep / discard). |
| **writing-skills** | Used when creating or editing skills, or verifying they work before deployment. |

## 2. How the skills chain together

The README lays out an explicit **Basic Workflow**, and the skills are designed to hand off to one another in this order:

1. **brainstorming** — activates before any code is written. Refines a rough idea through questions, classifies the request, explores alternatives, and (for larger work) saves a design document.
2. **using-git-worktrees** — activates after design approval. Creates an isolated workspace on a new branch and verifies a clean test baseline.
3. **writing-plans** — activates once the design is approved. Breaks the work into small, fully-specified tasks.
4. **subagent-driven-development** *or* **executing-plans** — activates once a plan exists. Either dispatches a fresh subagent per task (two-stage review: spec compliance, then code quality) or executes in batches with human checkpoints.
5. **test-driven-development** — activates during implementation of each task. Enforces write-failing-test → watch it fail → minimal code → watch it pass → commit; deletes any code written before its test.
6. **requesting-code-review** / **receiving-code-review** — activate between tasks, reviewing against the plan and reporting issues by severity; critical issues block progress.
7. **finishing-a-development-branch** — activates once all tasks are complete and tests pass; presents merge/PR/keep/discard options and cleans up the worktree.

Supporting/meta skills plug into this chain rather than sitting in the main line:
- **using-superpowers** is the entry point — it runs at the start of any conversation and is what causes the agent to check for a relevant skill (including brainstorming) before doing anything else, even asking clarifying questions.
- **dispatching-parallel-agents** is invoked whenever a step (often inside subagent-driven-development) contains 2+ independent tasks.
- **systematic-debugging** and **verification-before-completion** cut across the whole flow — the former whenever a bug/failure appears, the latter right before any "done" claim, commit, or PR.
- **writing-skills** is a meta-skill for extending the system itself (creating/editing skills), not part of the feature-delivery chain.

## 3. Origin, author, timeline, philosophy

- **Author**: Jesse Vincent (GitHub: `obra`; blog: blog.fsck.com), building it as part of **Prime Radiant** (primeradiant.com), which also offers commercial support for enterprise users of Superpowers.
- **Launch**: Original release announcement published **October 9, 2025** on blog.fsck.com (`blog.fsck.com/2025/10/09/superpowers/`). The GitHub repo `obra/superpowers` was created the same day (`created_at: 2025-10-09T19:45:18Z`, confirmed via GitHub API).
- **Distribution**: Available through Anthropic's **official Claude plugin marketplace** (`/plugin install superpowers@claude-plugins-official`), Anthropic's plugin listing page (claude.com/plugins/superpowers), and a separate community "Superpowers Marketplace" (`obra/superpowers-marketplace`). It has since expanded beyond Claude Code to other coding-agent harnesses (Antigravity, Codex App/CLI, Cursor, Devin CLI, Factory Droid, Gemini CLI, GitHub Copilot CLI, Grok Build CLI, Kimi Code, OpenCode, Pi, Hermes Agent), per the README's installation table of contents.
- **Stated philosophy** (from the README's Philosophy section):
  - **Test-Driven Development** — write tests first, always.
  - **Systematic over ad-hoc** — process over guessing.
  - **Complexity reduction** — simplicity as the primary goal (this is where **YAGNI** ["You Aren't Gonna Need It"] and **DRY** are invoked, per the README's "How it works" description of the planning stage).
  - **Evidence over claims** — verify before declaring success (the basis for the `verification-before-completion` skill).
  - The README also frames the overall approach as **spec-first**: the agent "doesn't just jump into trying to write code" — it teases out a spec through brainstorming, gets sign-off, and only then plans and implements.

## 4. Notable features

- **Brainstorming's spike / bounded / architectural classification** (added in v6.3.0, per RELEASE-NOTES: "Ceremony now scales to the task. Requests are classified as spike, bounded, or architectural; small tasks skip the two-document ritual. Every path still stops for your approval before implementation."). Per `skills/brainstorming/SKILL.md`:
  - **Spike** — a feasibility question ("can we…", "is it possible…"); terminal state is presenting a probe and getting a nod, then investigating and reporting a recommendation — no plan document.
  - **Bounded** — a well-scoped change to code that already exists in the repo (measured by whether there's an existing flow to change, not by the agent's familiarity with the app type); terminal state is the normal development workflow with no plan document, gated on the same hard approval as architectural work.
  - **Architectural** — new projects, new subsystems, or changes lacking an existing flow to modify; this is the only path that goes on to writing-plans, executing-plans, subagent-driven-development, mcp-builder, or other implementation skills.
  - The skill explicitly guards against gaming this classification (e.g., "I'll call it bounded and skip the spec" is called out in its rationalization table as a red flag rather than a shortcut).

- **Visual companion**: an optional local web server that brainstorming can open alongside the terminal conversation, for showing mockups/prototypes/interactive choices when the discussion involves visual content (added per RELEASE-NOTES history, e.g. `docs/plans/2026-01-17-visual-brainstorming.md`: "Give Claude a browser-based visual companion for brainstorming sessions - show mockups, prototypes, and interactive choices alongside terminal conversation"). It reports the installed Superpowers version and loads Prime Radiant's logo from their website by default; per the README's "Visual companion telemetry" section this is used only for rough usage/version analytics (no project/prompt/agent details, no click tracking) and can be disabled via the `SUPERPOWERS_DISABLE_TELEMETRY` env var (also honors Claude Code's own `DISABLE_TELEMETRY` / `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` opt-outs). A later release note (in the local RELEASE-NOTES.md) also mentions the companion was hardened with "a real security model" after it shipped without authentication.

- **Subagent-driven development**: dispatches a fresh subagent per plan task with two-stage review (spec compliance, then code quality). v6.3.0 release notes describe further hardening: controllers recording rulings on non-catastrophic plan conflicts instead of stalling (one donated session had reportedly sat blocked for ~9 hours on a question the controller could have decided), a pre-dispatch conflict scan, batching of small same-shape tasks to cut subagent cost, and a rule that implementers/reviewers may not spawn their own subagents (was producing duplicate reviews).

## Notable claims and reception **[UNVERIFIED figures noted]**

- GitHub repository `obra/superpowers`: **281,744 stargazers** and **25,232 forks** as of 2026-09-04 (verified directly via the GitHub API, `stargazers_count`/`forks_count` fields) — an unusually large following for a skills/plugin repo. A secondary blog source claimed "~248k stars as of July 2026" and "over 1 million installs"; the stars figure is roughly consistent with organic growth from July to September, but the **1 million installs figure is [UNVERIFIED]** — no primary source (GitHub, Anthropic, or Prime Radiant) was found confirming an install count.
- Simon Willison (simonwillison.net, "Superpowers: How I'm using coding agents in October 2025") called it "a really significant piece" and Jesse Vincent "one of the most creative users of coding agents" he knows, specifically highlighting the `systematic-debugging` skill's root-cause-tracing technique and Jesse's use of Graphviz DOT graphs as agent-readable workflow diagrams. Willison also relays Jesse's claim that the core framework is "very token light," pulling in a primer doc of under 2k tokens and offloading implementation work to subagents.
- Multiple third-party "best Claude Code plugins" round-up blogs (designrevision.com, mcp.directory, claudemarketplaces.com, augmentclaude.com, claudedigest.com) list Superpowers favorably, describing it as targeting AI coding agents' tendency toward "eagerness" (jumping to an answer instead of the right one) by imposing plan-first discipline. These are secondary SEO-style sources rather than primary reporting, so specific superlative claims from them (e.g., "the plugin worth installing") are **[UNVERIFIED]** beyond the general sentiment they represent.

## Sources

- Local install: `/Users/mohammedhussain/.claude/plugins/cache/claude-plugins-official/superpowers/6.3.0/` (`README.md`, `RELEASE-NOTES.md`, `package.json`, `skills/*/SKILL.md`, `docs/plans/2026-01-17-visual-brainstorming.md`)
- [Superpowers original release announcement — blog.fsck.com, 2025-10-09](https://blog.fsck.com/2025/10/09/superpowers/)
- [obra/superpowers — GitHub repository](https://github.com/obra/superpowers/)
- GitHub API: `https://api.github.com/repos/obra/superpowers` (stargazers_count, forks_count, created_at — fetched 2026-09-04)
- [Superpowers Plugin — Claude by Anthropic (official marketplace listing)](https://claude.com/plugins/superpowers)
- [Simon Willison — "Superpowers: How I'm using coding agents in October 2025"](https://simonwillison.net/2025/Oct/10/superpowers/)
- [Claude Code Superpowers Plugin: Guide & Setup (2026) — designrevision.com](https://designrevision.com/blog/claude-code-superpowers-plugin)
- [Superpowers for Claude Code: Still Worth It in 2026? — mcp.directory](https://mcp.directory/blog/superpowers-skill-worth-it-2026)
- [superpowers · obra/superpowers — claudemarketplaces.com](https://claudemarketplaces.com/plugins/obra-superpowers/superpowers)
- [Superpowers — AugmentClaude Claude Skills Marketplace](https://augmentclaude.com/s/superpowers-obra)
- [The Claude Code plugin worth installing - Super Powers — claudedigest.com](https://www.claudedigest.com/posts/fec89e28-30d8-4169-a57a-71d99e199628)
