# Presentation Outline (revised 2026-09-11, tracks ./presentation/index.html)

X by 2 tech talk, 15–20 minutes. General technical audience with AI/agentic-coding awareness but little to no hands-on exposure to agent skills. Purpose = awareness + enough practical grounding to try it. Tone: perspective from our own work, exploratory, not prescriptive.
   
## Structure

1. **Title** — Superpowers: Skills for AI-Driven Development.

2. **The shift** (~1 min)
   - AI agents now write most of the code in agent-assisted projects — generating code is cheap.
   - What's still expensive: deciding what to build, and verifying it was built right.
   - Frames the talk as a process question, not a "can AI code" question.

3. **SDD teaser** (~1 min)
   - Plants the spec-driven-development hypothesis (humans on specs/verification, agents on the tedious middle) as "a perspective we're forming," deliberately hedged.
   - Explicit callback planned for the closing.

4. **What is a skill? — concept** (~1.5 min)
   - Definition: a folder of instructions an agent pulls in when relevant — not pasted, not always-loaded.
   - Distinction from adjacent concepts this audience may already know: not a command (invoked explicitly) and not an MCP server (a tool the agent calls) — a skill is instructions the agent decides to pull in on its own.
   - No code on this slide; concept only.

5. **What is a skill? — how it works** (~2 min)
   - `SKILL.md` folder structure + frontmatter code example (name, description, required vs. optional fields).
   - Progressive disclosure, folded in here as the answer to "why this stays cheap": 3 stages — discovery (name+description only, ~100 tokens/skill), activation (full SKILL.md loads on match), execution (scripts/references load only if needed).
   - Payoff line: a large skill library doesn't have to cost much context if well-factored — this is the setup for the "skill bloat" debate later.

6. **Open standard, quickly** (~1.5 min)
   - Timeline: Oct 2025 (Anthropic announces Agent Skills for Claude) → Dec 2025 (spun out as vendor-neutral agentskills.io) → within days (OpenAI/Codex adds support) → 30+ agents today (Claude Code, Codex, Copilot, VS Code, Cursor, Gemini CLI, …).
   - Distribution note: plugin marketplaces, community catalogs, git repos.
   - Point: skills you write are portable, not a single-vendor gimmick.

7. **Enter Superpowers** (~2 min)
   - A plugin of 14 skills covering the full SDLC: brainstorm → plan → implement → verify → review → merge.
   - By Jesse Vincent (obra), launched Oct 2025, on Anthropic's official plugin marketplace, 280k+ GitHub stars.
   - Philosophy: spec-first, test-driven, YAGNI, "evidence over claims."
   - The agent doesn't jump to code — it teases out a spec, gets sign-off, then plans and implements.

8. **The skill map** (~2 min) — anchor visual
   - Dependency diagram: `using-superpowers → brainstorming → using-git-worktrees → writing-plans → subagent-driven-development / executing-plans → finishing-a-development-branch`, plus cross-cutting skills during implementation (TDD, systematic-debugging, verification-before-completion) and between tasks (code review, dispatching-parallel-agents, writing-skills).
   - One-clause gloss where `subagent-driven-development` is first highlighted: "(a subagent = a fresh, isolated agent instance dispatched to do one task and report back)"
   - Point: some skills hand off to each other in sequence, not ad-hoc.

9. **Brainstorming: ceremony scales to the task** (~1.5 min)
   - Three classifications: spike (probe, no documents), bounded (short design in chat, no plan doc), architectural (questions → approaches → written spec → plan).
   - Every path stops for approval before implementation regardless of ceremony level.
   - Optional meta-note: this deck's own outline/design went through this flow.

10. **The visual companion** (~1 min)
    - Optional local web UI opened alongside the terminal during brainstorming, for mockups/diagrams/side-by-side options instead of prose descriptions.
    - Offered just-in-time, only for genuinely visual questions.
    - Caveat flagged here, paid off in the risks section: token-intensive.

11. **From brainstorm to spec** (~1 min) — real artifact
    - Excerpt from an actual `docs/superpowers/specs/*-design.md` (goal, a requirement or two, one architecture decision) — **[content need: paste real excerpt, currently a placeholder in index.html]**.
    - Point: the spec is the durable, reviewed artifact — the thing worth version-controlling.

12. **Plans → subagent-driven development** (~1.5 min) — real artifact
    - Excerpt from a real implementation plan: exact file paths, 2–5 minute task granularity, verification steps — **[content need: placeholder in index.html]**.
    - Mechanism: a fresh subagent per task, two-stage review (spec compliance, then code quality); main session stays clean.

13. **Verification: evidence over claims** (~1 min) — real artifact
    - Excerpt from a real test report / verification output — **[content need: placeholder in index.html]**.
    - `test-driven-development` (failing test first, always) + `verification-before-completion` (no "it works" without command output).

14. **What we liked** (~1.5 min)
    - Structure (right-sized process per problem), approval gates (fewer confident wrong turns), visual companion (seeing beats reading), subagent leverage (strong model plans, cheaper agents execute in parallel).
    - Ground each bullet in a project anecdote when presenting.

15. **What to watch out for** (~1.5 min)
    - Model choice matters (small models fine for research, poor for development).
    - Token cost (thorough process = more input/output, especially the visual companion) — payoff of the slide-10 caveat.
    - Subagent ergonomics (dispatching isn't always the right call; the framework won't make that judgment for you).

16. **The "skill bloat" question** (~2 min)
    - The critique: instructions/tools eat context before work starts; tool-selection accuracy drops as options grow; "why pay tokens for harness on models that already plan competently?"
    - The counters: progressive disclosure makes a big library cheap if well-factored (callback to slide 5); consistency across a team is worth tokens; alternative is growing small, project-specific skills reactively from observed failures.
    - Framing line: critics mostly don't dispute the *workflow* (TDD, plan-first, review) — they dispute paying for it on every task.

17. **Our take, by where you are** (~1.5 min)
    - New/newish to AI-assisted dev → try brainstorming with Superpowers (teaches a good default process while you use it).
    - Experienced → try a spec-driven skill set; Superpowers is one, GitHub Spec Kit / Amazon Kiro / BMAD are named as other points in the category (names only, not elaborated — avoid a proper-noun dump here).
    - Advanced, with an existing skill system → don't adopt wholesale, but worth reviewing for ideas.

18. **Where this is heading** (~1 min)
    - Callback to slide 3: humans spend time on specs and verification, AI takes the tedious middle.
    - Explicitly exploratory framing — a perspective from our work, not a prescription.

19. **Try it** (~1 min) — closing / CTA
    - `/plugin install superpowers@claude-plugins-official`; agentskills.io; github.com/obra/superpowers.
    - Local call to action: our own `xby2-skills` library — contribute the skills our projects need.
    - Note: this deck itself was built with Claude Code + Superpowers brainstorming.

## Open items
- Three artifact placeholders (slides 11–13) still need real excerpts pasted in from an actual project spec/plan/test-report.
- Two accessibility insertions still need to land in `index.html`: the commands/MCP distinction (slide 4) and the subagent gloss (slide 8).
- Slide 17's other-SDD-tools mention stays name-only by design; do not expand into a comparison table in the talk itself (reference material lives in [spec-driven-landscape.md](spec-driven-landscape.md) if asked in Q&A).

## Deliverable
reveal.js HTML deck, served via GitHub Pages from `gh-pages` branch. Current working copy: `./presentation/index.html`.
