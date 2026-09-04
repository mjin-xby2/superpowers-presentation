# Presentation Outline (approved 2026-09-04)

X by 2 tech talk, 15–20 minutes. General technical audience; purpose = awareness + enough practical grounding to try it. Tone: perspective from our own work, exploratory, not prescriptive.

## Structure

1. **Opening: the shift we're watching** (~2 min)
   - AI writes code well; bottleneck moving to deciding what to build + verifying it.
   - Light touch on spec-driven development — "a perspective we're forming."
2. **What are skills?** (~3 min)
   - SKILL.md format, progressive disclosure, agentskills.io open standard.
   - Timeline: Anthropic Oct 2025 → open standard Dec 2025. Broad agent support. Marketplaces.
   - See [agentskills-standard.md](agentskills-standard.md).
3. **Enter Superpowers** (~2 min)
   - Skill collection covering the full SDLC; Jesse Vincent; TDD/YAGNI/spec-first philosophy.
   - See [superpowers.md](superpowers.md).
4. **Tour of the skill map** (~5 min)
   - Anchor visual: dependency diagram (brainstorming → writing-plans → executing-plans / subagent-driven-development → supporting skills).
   - Brainstorming's spike/bounded/architectural classification; visual companion; subagent-driven development.
   - Real artifact excerpts: spec, implementation plan, test report (provided by presenters).
5. **What we've experienced** (~4 min)
   - Benefits: structured approach, visual companion, subagent leverage.
   - Risks: model choice matters, token cost, skill bloat (framed via token/tool-selection-accuracy argument + "grow your own skills reactively" alternative, per [skill-bloat-critique.md](skill-bloat-critique.md)), clunky subagent ergonomics.
6. **Closing** (~2 min)
   - Tiered recommendation: new → try Superpowers brainstorming; experienced → try a spec-driven skill set (Superpowers, Spec Kit, Kiro, BMAD — see [spec-driven-landscape.md](spec-driven-landscape.md)); advanced → mine it for ideas.
   - Future-of-development perspective (humans on specs + artifacts, AI on tedium) — explicitly exploratory.
   - Local application: xby2-skills.

## Deliverable
reveal.js HTML deck, served via GitHub Pages from `gh-pages` branch.
