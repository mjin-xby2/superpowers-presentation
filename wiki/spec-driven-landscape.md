# Spec-Driven Development Landscape (2026)

**One-line summary:** "Spec-driven development" (SDD) — writing structured specs before code so AI agents build against agreed intent rather than vibes — has become a crowded, fast-moving category through 2026 (GitHub Spec Kit, Amazon Kiro, BMAD Method, Agent OS, OpenSpec, Tessl, and more), all competing with plain "plan mode" and with critics calling it warmed-over waterfall; Superpowers takes a different tack, distributing durable *process skills* (TDD, debugging, code review discipline) rather than a spec-artifact pipeline.

---

## 1. GitHub Spec Kit (github/spec-kit)

**What it is:** An open-source toolkit from GitHub for Spec-Driven Development. It is MIT-licensed and maintained by GitHub; it reached v1.0.0 in 2026 after roughly a year of public development (v1.0.1 shipped August 2026).

**How it works:** Spec Kit imposes a six-stage pipeline on any AI coding agent: **Constitution** (project principles/standards) → **Specify** (natural-language requirements) → **Plan** (technical architecture/tech choices) → **Tasks** (broken-down implementation items) → **Implement** (agent executes tasks) → **Converge** (validate finished work against the spec and refine). It is agent-agnostic — the project claims support for 30+ coding agents/CLIs, including GitHub Copilot, Claude Code, Cursor, Gemini CLI, Codex CLI, Windsurf, and others. It is explicitly positioned against "vibe coding" (vague prompt → hope for the best): the spec, not the prompt, is meant to be the durable artifact.

**Traction claim [UNVERIFIED]:** Some community write-ups cite "60–80% fewer rework cycles" versus prompt-driven development; this is an anecdotal/marketing-style claim from secondary blog sources, not a controlled study.

**vs. Superpowers:** Spec Kit is a rigid, sequential SDLC pipeline enforced through slash-command-like phases and generated spec/plan/task documents that live in the repo as artifacts. Superpowers is a library of independent, composable *skills* (debugging loops, TDD discipline, code-review checklists, git-worktree isolation) invoked situationally rather than a fixed phase gate — there is no single mandated "constitution → converge" pipeline, and no single canonical spec document is produced as the organizing artifact.

---

## 2. Amazon Kiro

**What it is:** Kiro is an agentic AI coding IDE — a VS Code (Code OSS) fork — built and operated by AWS. It launched in public preview in mid-2025 and has continued to develop through 2026, including a "Kiro Crew" free/open-source workspace variant, web and mobile interfaces, and enterprise features (IAM/SSO). It is a full product/service, not a plugin or file-based method.

**How it works:** Developers describe requirements in natural language; Kiro converts this into **specs** — user stories with acceptance criteria written in EARS format (Easy Approach to Requirements Syntax), a technical design document (including data models, API endpoints, sequence/architecture diagrams), and a sequenced task list. Developers review and approve before Kiro executes tasks, one step (or a batch) at a time, using parallel agents; it also generates property-based tests, keeps docs in sync with API changes, and runs security/secret scans on commits. It is powered by Claude models (Sonnet 3.7/4.0 cited at launch/through 2026) with an "Agent Model Selection" capability. Pricing is credit-based with paid tiers.

**vs. Superpowers:** Kiro is a vendor product (AWS) with a built-in IDE, hosted execution, and a proprietary spec-generation format (EARS) baked into the tool itself — you adopt Kiro's environment. Superpowers is a lightweight, model-agnostic plugin of markdown skill files that layer onto whatever agent/CLI the user already runs (e.g., Claude Code); there's no IDE fork, no hosted service, and no formal requirements-syntax standard like EARS.

---

## 3. BMAD Method (BMAD-METHOD™)

**What it is:** "Breakthrough Method for Agile AI-Driven Development" — an open-source (MIT) framework created and maintained by BMad Code, LLC (GitHub org `bmad-code-org`), distributed via npm (`bmad-method`) and as a set of files (agent persona definitions in Markdown/YAML, workflows, templates, CLI installer) that drop into an existing AI coding environment. It has a large community footprint (tens of thousands of GitHub stars) and an active Discord.

**How it works:** BMAD assigns the AI agent a rotating cast of personas — analyst, product manager, architect, developer, tester, "scrum master," etc. — each with distinct commands and artifact outputs. It runs in two phases: **Agentic Planning** (multi-agent discussion produces PRD/architecture artifacts, making decisions and trade-offs explicit before code) and **Context-Engineered Development** (dev/QA agents implement against that full planning context, reducing ambiguity). The overall loop is Plan → Build & Verify → Learn & Adjust, and the framework claims to be "right-sized," letting small changes skip straight to implementation.

**vs. Superpowers:** BMAD's central metaphor is a simulated *agile team of role-playing agents* moving through a formal planning-then-building lifecycle — heavier-weight and closer to a scaled-agile/SDLC simulation. Superpowers doesn't role-play a team; it hands the single acting agent a toolbox of skills to invoke as needed (systematic-debugging, TDD, brainstorming, code review) and leans on the human + one agent working directly, without multi-persona handoffs or generated PRD/architecture documents as required gates.

---

## 4. Other notable spec-driven / SDLC-skill frameworks (2025–2026)

- **Agent OS** (buildermethods/agent-os, created by Brian Casel of Builder Methods): A system for injecting a team's own codebase standards into spec-driven workflows rather than prescribing a universal spec format. v2/v3 (2026) increasingly defers to each agent's native "Plan Mode" (Claude Code, Cursor, Antigravity) rather than owning the whole pipeline, adding a `/shape-spec` command that asks targeted questions informed by the team's documented standards and product mission. It's positioned as a lighter-weight layer than Spec Kit/BMAD — closer in spirit to a skills/standards add-on than a full method.

- **OpenSpec** (Fission-AI/openspec and forks): A lightweight, open-source SDD framework separating "specs" (`openspec/specs/` — current source of truth) from "changes"/proposals (`openspec/changes/` — structured change requests with technical designs, task checklists, and archives of completed changes). Claims 30+ tool integrations (Claude Code, Cursor, Copilot, etc.). Positioned as more minimal/diff-friendly than Spec Kit.

- **Tessl**: A commercial, MCP-native, agent-agnostic platform (not tied to one IDE) that installs as "tiles" into a project's `.tessl/` directory, teaching any MCP-compatible agent (Claude Code, Cursor, Copilot, Gemini) a spec-driven workflow from outside the agent itself. Markets audit trails for regulated industries. **[UNVERIFIED]** exact funding/pricing details beyond what's on its own site.

- **Google Antigravity** and native **"Plan Mode"** features in Cursor and Claude Code are frequently cited alongside these frameworks in 2026 comparisons — the general industry pattern by mid-2026 is that "every major AI coding tool… has shipped its own flavor of SDD" (Spec Kit, Kiro, Cursor, OpenSpec, BMAD, Tessl, Antigravity), such that dedicated third-party frameworks increasingly compete with vendors' own built-in planning modes. **[UNVERIFIED]** — this framing comes from secondary blog/aggregator sources (e.g., dev.to comparison posts), not the vendors themselves.

- No direct "OpenAI Codex spec-driven skill framework" equivalent to Spec Kit/BMAD was found as a named, packaged product; OpenAI's contribution to this conversation is intellectual/cultural rather than tooling — see Sean Grove's talk below — plus OpenAI's own internally-facing "Model Spec" document, which Grove cites as an exemplar of spec-as-source-of-truth.

---

## 5. The industry conversation: is SDD gaining traction?

**The enthusiastic case.** The catalytic moment usually cited is Sean Grove's (OpenAI) talk **"The New Code,"** delivered at the AI Engineer World's Fair in 2025. His thesis: code is a lossy projection of human intent, and in an era where agents write the code, the scarce and valuable skill is *structured communication of intent* — i.e., writing specifications — not typing syntax. He argues the code itself is only 10–20% of a programmer's value; the other 80–90% is understanding the problem, distilling requirements, and verifying the right thing got built. He points to OpenAI's own **Model Spec** — a living natural-language document legible to non-engineers (product, legal, safety, policy) who can debate and contribute to it like source code — as the model for what a "spec" should be: durable, versioned, and treated as the actual source of truth, in contrast to developers who "shred the source [prompts] and very carefully version control the binary [output]." This talk is widely referenced across 2026 SDD commentary as the intellectual anchor for the movement, and by mid-2026 essentially every major AI coding vendor had shipped some SDD variant (see section 4), which enthusiasts read as validation.

**The skeptical case.** Critics argue SDD is old wine in a new bottle. Brandon Kindred's piece **"Same Patterns, New Hype"** (2026) argues SDD is largely waterfall/contract-design methodology rebranded for the agent era, and that "the value is the thinking you do while writing the spec, not the tooling around it" — i.e., the ceremony (constitutions, converge phases, persona rotations) may be theater around a genuinely useful but simple discipline (think before you code). Others frame the skepticism more bluntly as "the return of waterfall: listen to the PM and write requirement documents," worrying SDD reintroduces heavyweight upfront-design friction that agile explicitly reacted against. There is also a live debate about autonomy: some practitioners argue that by mid-2026, frontier coding agents can execute the Spec Kit-style steps (spec → plan → tasks → implement) autonomously, cheaply, and well enough that human review gates at each phase are unnecessary friction; the counterargument (echoed in enterprise-context commentary) is that human gates in the SDLC were never purely about agent capability — they exist because of "risk management, compliance, legal accountability, governance, [and] business responsibility," constraints an agent's proficiency doesn't remove.

**The synthesis framing that recurs across 2026 sources:** SDD is presented as a direct structural response to "drift" — confident, plausible-looking AI-generated code that quietly solves the wrong problem because nobody grounded the work in a real, checkable specification — a failure mode common to unstructured "vibe coding." Whether the correct fix is a formal, tool-enforced pipeline (Spec Kit, Kiro, BMAD) or a lighter-weight discipline layered onto existing plan-mode features (Agent OS v3, native Cursor/Claude Code plan mode) is the crux of the 2026 debate, and is also the crux of where Superpowers positions itself: closer to the "lightweight discipline via skills" end of the spectrum than the "formal pipeline with generated spec artifacts" end.

---

## Sources

- [GitHub Spec Kit documentation](https://github.github.com/spec-kit/)
- [github/spec-kit repository](https://github.com/github/spec-kit)
- [GitHub's Spec Kit Puts the Spec Back in Software Development — DevOps.com](https://devops.com/githubs-spec-kit-puts-the-spec-back-in-software-development/)
- [What Is GitHub Spec Kit? — knightli.com](https://knightli.com/en/2026/05/25/github-spec-kit-spec-driven-development/)
- [Complete Guide to GitHub Spec Kit (2026) — Dailyaiworld](https://dailyaiworld.com/blogs/spec-driven-development-github-spec-kit-2026)
- [Meet GitHub Spec-Kit — MarkTechPost](https://www.marktechpost.com/2026/05/08/meet-github-spec-kit-an-open-source-toolkit-for-spec-driven-development-with-ai-coding-agents/)
- [GitHub Spec Kit Review (2026) — vibecoding.app](https://vibecoding.app/blog/spec-kit-review)
- [Kiro official site](https://kiro.dev/)
- [Introducing Kiro: Amazon's Spec-Driven, Agentic IDE — Medium](https://medium.com/@ygsh0816/introducing-kiro-amazons-spec-driven-agentic-ide-dc597900dc9f)
- [Kiro Agentic AI IDE — AWS re:Post](https://repost.aws/articles/AROjWKtr5RTjy6T2HbFJD_Mw/%F0%9F%91%BB-kiro-agentic-ai-ide-beyond-a-coding-assistant-full-stack-software-development-with-spec-driven-ai)
- [Kiro Review 2026 — heyuan110.com](https://www.heyuan110.com/posts/ai/2026-03-10-kiro-review/)
- [Kiro IDE Review (2026) — Ricardo Gil](https://www.gilricardo.com/blog/kiro-review-spec-driven-agentic-ide)
- [AWS Launches Kiro — Forbes](https://www.forbes.com/sites/janakirammsv/2025/07/15/aws-launches-kiro-a-specification-driven-agentic-ide/)
- [AWS Kiro, Agentic Coding and the Rise of Spec-Driven AI Development — dev.to](https://dev.to/aws-builders/aws-kiro-agentic-coding-and-the-rise-of-spec-driven-ai-development-41h)
- [aws kiro spec driven agent — InfoQ](https://infoq.com/news/2025/08/aws-kiro-spec-driven-agent)
- [bmad-code-org/BMAD-METHOD repository](https://github.com/bmad-code-org/BMAD-METHOD)
- [The BMAD Method: A Framework for Spec Oriented AI-Driven Development — GMO](https://recruit.group.gmo/engineer/jisedai/blog/the-bmad-method-a-framework-for-spec-oriented-ai-driven-development/)
- [What Is the BMAD Method? — Augment Code](https://www.augmentcode.com/guides/bmad-method-ai-development)
- [A Comparative Analysis: BMAD-Method vs. GitHub Spec Kit — Medium](https://medium.com/@mariussabaliauskas/a-comparative-analysis-of-ai-agentic-frameworks-bmad-method-vs-github-spec-kit-edd8a9c65c5e)
- [BMAD vs. Spec-Driven Development — martinelli.ch](https://martinelli.ch/bmad-vs-spec-driven-development-why-ai-needs-better-specifications/)
- [What is BMAD-METHOD™? — Medium](https://medium.com/@visrow/what-is-bmad-method-a-simple-guide-to-the-future-of-ai-driven-development-412274f91419)
- [The BMAD Method Explained: Multi-Agent AI Agile — codemyspec.com](https://codemyspec.com/blog/bmad-method-explained)
- [buildermethods/agent-os repository](https://github.com/buildermethods/agent-os)
- [Agent OS v2 — buildermethods.com](https://buildermethods.com/agent-os/v2)
- [Spec-Driven Development with Agent OS workshop — buildermethods.com](https://buildermethods.com/sessions/spec-driven-development-workshop)
- [Agent OS v2 Demo — buildermethods.com](https://buildermethods.com/library/agent-os-v2-demo)
- [Fission-AI/openspec repository](https://github.com/Fission-AI/openspec)
- [awesome-openspec — GitHub](https://github.com/wearetechnative/awesome-openspec)
- [OpenSpec — intent-driven.dev](https://intent-driven.dev/knowledge/openspec/)
- [Spec-Driven Development with OpenCode and OpenSpec — nsclass.github.io](https://nsclass.github.io/2026/05/24/spec-driven-development-openspec-opencode)
- [I Tested the Top Spec-Driven Dev Tools in 2026 — dev.to](https://dev.to/filiksyos/i-tested-the-top-spec-driven-dev-tools-in-2026-4gdm)
- [9 Best AI Tools for Spec-Driven Development in 2026 — MarkTechPost](https://www.marktechpost.com/2026/05/08/9-best-ai-tools-for-spec-driven-development-in-2026-kiro-bmad-gsd-and-more-compare/)
- [Tessl Review 2026 — codemyspec.com](https://codemyspec.com/blog/tessl-review)
- [Best context engineering tools for AI coding in 2026 — packmind.com](https://packmind.com/context-engineering-ai-coding/best-context-engineering-tools/)
- [The New Code — Sean Grove, OpenAI (transcript) — lawwu.github.io](https://lawwu.github.io/transcripts/8rABwKRsec4.html)
- [The New Code — Sean Grove, OpenAI (summary) — my.infocaptor.com](https://my.infocaptor.com/hub/summaries/ai-engineer/the-new-code-sean-grove-openai-8rABwKRsec4)
- [The End of Coding? How Specifications Are Becoming the New Source Code — implicator.ai](https://www.implicator.ai/the-end-of-coding-how-specifications-are-becoming-the-new-source-code/)
- [The Most Valuable Developer Skill in 2025? Writing Code Specifications — tessl.io](https://tessl.io/blog/the-most-valuable-developer-skill-in-2025-writing-code-specifications/)
- [The Rise of Spec-Driven Development — Frontend at Scale](https://frontendatscale.com/issues/49/)
- [The Future of Programming is Writing Better Instructions, Not Better Code — ikangai.com](https://www.ikangai.com/the-future-of-programming-is-writing-better-instructions-not-better-code/)
- [agentic-engineering-field-study: spec-driven-development.md — GitHub](https://github.com/ianhxu/agentic-engineering-field-study/blob/main/04-spec-driven-development.md)
- [Spec-Driven Development for AI Agents: Governing Specs — truefoundry.com](https://www.truefoundry.com/blog/spec-driven-development-ai-agents)
- [Spec-Driven Development in 2026 — dev.to](https://dev.to/krlz/spec-driven-development-in-2026-what-it-is-the-tooling-and-how-teams-actually-use-it-2fk2)
- [From Prompt to Process: a Process Taxonomy... — arXiv](https://arxiv.org/pdf/2606.04967)
- [Spec-Driven Development (SDD): The Definitive 2026 Guide — thebcms.com](https://www.thebcms.com/blog/spec-driven-development/)
- [Spec-Driven Development with AI Coding Agents (2026) — tryzeroshot.com](https://tryzeroshot.com/blog/spec-driven-development-with-ai-coding-agents)
- [Spec-Driven Development: A Spec-First Approach to AI-Native Engineering — Microsoft for Developers](https://developer.microsoft.com/blog/spec-driven-development-ai-native-engineering/)
- [What Is Spec-Driven Development? A Complete Guide — Augment Code](https://www.augmentcode.com/guides/what-is-spec-driven-development)
