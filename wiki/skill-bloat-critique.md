# Skill Bloat: The Critique of Large Prescriptive Skill Collections

**One-line summary:** The consensus critique of frameworks like Superpowers is not that the workflow is *wrong*, but that it's *expensive* — large prescriptive skill/instruction sets consume context tokens and degrade tool-selection accuracy, so the emerging counter-recommendation is to build small, evidence-driven, project-specific skills instead of adopting big general-purpose collections wholesale.

---

## 1. The "bloated" critique of Superpowers-style collections

Superpowers (built by Jesse Vincent, accepted into Anthropic's official Claude Code marketplace) is a plugin that installs a structured set of skills (brainstorming, TDD, systematic debugging, subagent-driven development, plan-then-implement, etc.) to make Claude behave like a disciplined engineer following a fixed process.

- **Hacker News discussion** ("A Rave Review of Superpowers (For Claude Code)", https://news.ycombinator.com/item?id=47623101) surfaced several concrete criticisms:
  - User `tao_oat`: the spec/plan separation is often redundant — "the models are good enough to get straight to coding once the spec is written," and generated plans can devolve into throwaway markdown rather than adding value.
  - User `JohnCClarke`: questioned whether the heavy, structured process is aimed more at less-experienced developers/managers than seasoned engineers who don't need the scaffolding.
  - User `Syzygies`: argued for finer-grained, hands-on involvement than Superpowers' automated orchestration — working in smaller steps surfaces when *less* code is the right answer, which a big automated framework may not.
  - User `d--b`: reported Claude making *more* mistakes with Superpowers active than without it in some cases.
  - **[UNVERIFIED]** A commonly repeated framing — "the top-voted criticism is 'bloated,' not 'wrong' … nobody serious disputes the workflow, they dispute paying tokens for harness on models that plan competently unprompted" — appeared in aggregated secondary sources (e.g. mcp.directory, joanmedia.dev) rather than a single primary quote I could directly attribute to one named individual; treat as a paraphrase/synthesis of the HN discussion rather than a verified direct quote.

- **The token-cost framing** is made explicit in a companion piece, "The Honest Tradeoffs of Superpowers: Token Costs, Overkill, and the Alternatives" (joanmedia.dev) and in commentary describing Superpowers as making "Claude disciplined, not smarter" — whether that trade pays off depends on task size; solo/experienced developers are advised to cherry-pick modules (e.g., just brainstorm/plan) rather than install the full framework, while teams/newer users may benefit from the whole thing. **[UNVERIFIED — could not confirm original author/outlet reliability for this framing beyond secondary aggregator blogs.]**

## 2. The token-cost / context-window argument (with data)

This is the most concretely sourced part of the critique.

- **sph.sh, "Why Copying Claude Code Skills Doesn't Work"** (https://sph.sh/en/posts/why-copying-claude-code-skills-doesnt-work/): lays out token math — a "bloated setup" with 8+ copied MCP servers/skills can consume ~82K tokens on tool/skill definitions alone, leaving only ~27.5% of context free versus ~73% free in a minimal setup. Direct quote: *"The bloated setup loses 91K tokens to overhead. That is nearly half the context window consumed before the conversation starts."* The post cites TaskBench results showing tool-selection accuracy falling from 96% (1 tool) to 25% (8 tools), and references an Anthropic evaluation finding ~49% accuracy with 50+ unoptimized tools.
  - Its recommendation: start from zero, add a skill/MCP server only after a friction point recurs 5+ times, keep total configuration overhead under ~20% of the context window (checked via Claude Code's `/context` command), and adapt shared skills to your own conventions rather than importing them wholesale.

- **Armin Ronacher** ("Agentic Coding Recommendations," lucumr.pocoo.org, https://lucumr.pocoo.org/2025/6/12/agentic-coding/): argues for lean tooling generally — fast, simple scripts/Makefiles over elaborate frameworks; "have the agent do 'the dumbest possible thing that will work'"; prefers plain SQL over ORMs, boring stable languages (Go) over "magic"-heavy ecosystems, because agents get confused by complexity and churn. This is a general minimalism argument (not skills-specific per se) that underlies his later, more skills-specific writing.
  - In his newer post **"Pi: The Minimal Agent Within OpenClaw"** (https://lucumr.pocoo.org/2026/1/31/pi/), Ronacher (working with Mario Zechner's `pi` agent) describes a philosophy of radical simplicity: four tools only (Read, Write, Edit, Bash), the shortest system prompt of any agent, and letting the agent write its own tools/skills on demand rather than pre-loading a large curated library. He reportedly keeps only the skills he actually needs and "throws skills away if he doesn't need them" (per secondary summary — **[UNVERIFIED direct quote]**, but the "keep it minimal, prune unused skills" position is consistent across his posts).

- **Mario Zechner** (creator of the `pi` coding agent, `pi-mono`): per multiple summaries (dev.to, minai.dev, antoinebuteau.com), his system prompt + tool spec together fit in well under 1,000 tokens, versus the many-thousand-token instruction sets of heavier agents/frameworks. His stated principle (as paraphrased across sources): smaller prompts, fewer tools, more observability, plain-text `AGENTS.md` for customization instead of hidden plan modes or injected process scaffolding. Treats "context as the real interface" — the thing to protect, not spend. **[Could not locate a single canonical Zechner essay to quote verbatim; this is a synthesis of secondary write-ups about his `pi` project.]**

- **Simon Willison**: reportedly called Claude Skills "maybe a bigger deal than MCP" **[UNVERIFIED — found only in secondary aggregator summaries, not a directly fetched primary Willison post; his own site tag page (simonwillison.net/tags/armin-ronacher/) confirms he tracks and comments on Ronacher's minimalism-adjacent writing, but I did not verify the exact skills quote against his blog directly]**. This should be verified against simonwillison.net before use in a slide.

## 3. Steelman: arguments FOR structured skill collections

- **Discipline over intelligence, for repeatable quality on teams.** Superpowers proponents argue the framework's value isn't raw capability but consistency: a fixed brainstorm → plan → implement → verify loop, plus mandatory code review and TDD skills, catches classes of mistakes that an unprompted model — even a very capable one — will still make under time pressure or ambiguous scope. Per HN-discussion synthesis, the workflow itself is rarely disputed as *wrong*; what's disputed is whether paying the token cost is worth it for every task and every user.
- **Best for teams and less-experienced users; least valuable for expert solo devs on small tasks** — the recurring nuance across secondary commentary (mcp.directory, joanmedia.dev) is that full adoption suits teams/newcomers who benefit from codified best practice, while experienced solo developers are better off cherry-picking a few modules (e.g., brainstorming, planning) and letting the model handle routine execution unprompted. **[UNVERIFIED as a precise attributed claim — synthesized from multiple aggregator blog posts rather than one primary source.]**
- **The author's own counter to "bloat":** Jesse Vincent's post on building Superpowers (blog entry referenced in aggregator coverage) describes deliberately mining thousands of past conversation logs for candidate skills but choosing *not* to publish most of them (citing IP and "too esoteric" concerns), and describes testing whether a new skill is "comprehensible, complete" before merging it, plus using subagents for skill search specifically so "fruitless searches don't pollute the context window" — i.e., the project claims to apply its own anti-bloat discipline internally, rather than being an indiscriminate dump of process. Source: blog.fsck.com/2025/10/09/superpowers/ (Jesse Vincent's own blog, fsck.com).
- **Anthropic's own design intent argues bloat is avoidable by construction**, not an inherent property of "a skill collection" — see progressive disclosure below. If skills are authored well (metadata-only until relevant, full body only when triggered, linked reference files only when needed), a *large* library need not cost much context at rest; the failure mode is authors writing bloated individual SKILL.md files or systems that don't discriminate on relevance, not the existence of many skills per se.

## 4. Anthropic's own guidance on writing effective skills

Source: Anthropic engineering blog, "Equipping Agents for the Real World with Agent Skills" (anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills), plus the public best-practices doc bundled in the Superpowers repo itself (github.com/obra/superpowers/blob/main/skills/writing-skills/anthropic-best-practices.md).

- **Progressive disclosure is the central architectural principle**: Claude Code only loads a skill's *name + description* metadata at startup; it reads the full `SKILL.md` body only when a task looks relevant; it opens linked reference files/scripts only if the task demands that depth. This is explicitly designed so a large *library* of skills doesn't have to cost much context — only what's actually used gets loaded.
- **Start from observed gaps, not speculation**: "identify specific gaps in your agents' capabilities by running them on representative tasks," then build/refine skills to fill exactly those gaps — an empirical, incremental approach rather than writing a comprehensive process up front.
- **Split unwieldy skills**: when a `SKILL.md` grows too large, split content into separate files and link to them; keep "mutually exclusive or rarely used" content out of the main file to reduce what gets pulled into context by default.
- **Descriptions are the selection mechanism, not a place for detail**: a skill's `description` field should work like a table-of-contents entry, raising the odds Claude picks the right skill — not cramming execution detail into the metadata that's always loaded.
- **Iterative refinement from real usage**: "ask Claude to capture its successful approaches and common mistakes into reusable context and code" — i.e., let skills emerge from what actually happens in sessions, echoing Hashimoto's "harness engineering" idea below.
- Net effect: Anthropic's own guidance is squarely on the "lean, evidence-driven, well-factored" side of this debate — it doesn't argue against having many skills, but against *poorly factored* ones that ignore progressive disclosure.

## 5. Adjacent perspectives on lean, project-specific instructions

- **Mitchell Hashimoto**, "My AI Adoption Journey" (mitchellh.com/writing/my-ai-adoption-journey; also covered by Simon Willison and Pragmatic Engineer newsletter): describes "harness engineering" — maintaining a project's `AGENTS.md` reactively, adding one line only after an agent makes a real mistake, plus building small verification scripts (screenshot tools, filtered test runners) so the agent can self-check. Quote: *"Each line in that file is based on a bad agent behavior, and it almost completely resolved them all."* This is an evidence-first, minimal-and-growing approach to instructions — the opposite of pre-loading a large generic process — though Hashimoto isn't writing specifically about Anthropic's Skills feature or Superpowers.
- **AGENTS.md vs. Skills** (CircleCI blog, circleci.com/blog/agents-md-vs-skills): frames `AGENTS.md` as best for guidance that is "small, stable, and applies to nearly every task" (almost every project should have one), while Skills are best reserved for guidance that is "large, situational, or procedural" — implicitly arguing against stuffing everything into one always-loaded document, and for factoring bigger process only into on-demand skills.
- **Thorsten Ball**'s well-known "How to Build an Agent" tutorial (ampcode.com/notes/how-to-build-an-agent) demonstrates a fully working coding agent in under 400 lines of Go with a minimal tool set — not a direct statement about skill bloat, but consistent with the broader minimalist-agent-design current these critiques draw on. **[No direct Thorsten Ball statement about skill/instruction bloat specifically was found — flagging so this isn't overstated on a slide.]**

## 6. Bottom line for the presentation

- Nobody serious argues the *workflows* encoded in something like Superpowers are bad practice (TDD, plan-then-implement, code review are broadly endorsed).
- The dispute is about **cost vs. benefit at the token/context level**, and about **audience fit**: teams and less experienced users may benefit from a large codified process; experienced individual developers on small tasks often don't need to pay for it every time.
- Anthropic's own architecture (progressive disclosure) is explicitly designed to let large skill libraries avoid the bloat problem *if* authored well — the "bloat" critique is really a critique of unfactored, always-loaded instructions, not of having many skills in a repo.
- The practical synthesis recommended across sources: measure context usage, add skills/instructions only in response to observed, repeated failures (Hashimoto's "harness engineering," Anthropic's "start from evaluated gaps," sph.sh's "5+ times" rule), and prefer small project-specific skills over importing someone else's large general-purpose collection wholesale.

---

## Sources

- HN: "A Rave Review of Superpowers (For Claude Code)" — https://news.ycombinator.com/item?id=47623101
- HN: "Superpowers 6" — https://news.ycombinator.com/item?id=48739459
- sph.sh, "Why Copying Claude Code Skills Doesn't Work" — https://sph.sh/en/posts/why-copying-claude-code-skills-doesnt-work/
- Jesse Vincent, "Superpowers: How I'm using coding agents in October 2025" — https://blog.fsck.com/2025/10/09/superpowers/
- Armin Ronacher, "Agentic Coding Recommendations" — https://lucumr.pocoo.org/2025/6/12/agentic-coding/
- Armin Ronacher, "Pi: The Minimal Agent Within OpenClaw" — https://lucumr.pocoo.org/2026/1/31/pi/
- Armin Ronacher, "Agent Design Is Still Hard" — https://lucumr.pocoo.org/2025/11/21/agents-are-hard/
- Mitchell Hashimoto, "My AI Adoption Journey" — https://mitchellh.com/writing/my-ai-adoption-journey
- Simon Willison, coverage of Hashimoto's post — https://simonwillison.net/2026/Feb/5/ai-adoption-journey/
- Simon Willison, tag page tracking Armin Ronacher — https://simonwillison.net/tags/armin-ronacher/
- Anthropic engineering blog, "Equipping Agents for the Real World with Agent Skills" — https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- Anthropic best-practices doc (as bundled in the Superpowers repo) — https://github.com/obra/superpowers/blob/main/skills/writing-skills/anthropic-best-practices.md
- Claude blog, "Introducing Agent Skills" — https://claude.com/blog/skills
- Anthropic docs, "Agent Skills - Overview" — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- Thorsten Ball, "How to Build an Agent" — https://ampcode.com/notes/how-to-build-an-agent
- CircleCI blog, "AGENTS.md vs. skills: How to steer a coding agent" — https://circleci.com/blog/agents-md-vs-skills/
- Secondary/aggregator coverage (lower confidence, used only for framing, flagged inline as [UNVERIFIED] where a primary source could not be confirmed): mcp.directory/blog/superpowers-skill-worth-it-2026, joanmedia.dev/ai-blog/the-honest-tradeoffs-of-superpowers-token-costs-overkill-and-the-alternatives, dev.to/wonderlab (pi-mono coverage), minai.dev/posts/pi-coding-agent-lessons, antoinebuteau.com/lessons-from-mario-zechner
