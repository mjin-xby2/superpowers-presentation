# Agent Skills / agentskills.io

Agent Skills are a lightweight, open, portable file format (`SKILL.md` + optional bundled files) that lets any AI coding agent load specialized instructions, scripts, and resources on demand via "progressive disclosure"; Anthropic created the format for Claude in October 2025 and open-sourced it as a vendor-neutral standard (agentskills.io) in December 2025, since adopted by 30+ agent products including Claude Code, OpenAI Codex, Gemini CLI, GitHub Copilot, Cursor, and VS Code.

## 1. What is a "skill"? (SKILL.md format)

A skill is a directory containing, at minimum, a `SKILL.md` file:

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation loaded on demand
├── assets/           # Optional: templates, images, data files
└── ...
```

`SKILL.md` = YAML frontmatter + a Markdown instructions body.

**Frontmatter fields** (per the agentskills.io specification, https://agentskills.io/specification):

| Field | Required | Constraints |
|---|---|---|
| `name` | Yes | Max 64 chars; lowercase letters, digits, hyphens only; no leading/trailing/consecutive hyphens; must match the parent directory name |
| `description` | Yes | Max 1024 chars, non-empty; should state both what the skill does and when to use it, with keywords the agent can match against |
| `license` | No | License name or reference to a bundled license file |
| `compatibility` | No | Max 500 chars; environment requirements (target product, system packages, network access) |
| `metadata` | No | Arbitrary string→string map for client-specific extensions |
| `allowed-tools` | No | Space-separated list of pre-approved tools the skill may invoke (marked "Experimental" — support varies by agent) |

The Markdown body has no required structure; the spec recommends step-by-step instructions, input/output examples, and edge cases, and recommends keeping the main `SKILL.md` under ~500 lines / 5000 tokens, pushing detail into `references/`.

**Progressive disclosure** — the mechanism that keeps many skills cheap to keep "on hand" — works in three stages per the spec:
1. **Discovery**: at startup, only `name` + `description` (~100 tokens per skill) load into context for every available skill.
2. **Activation**: when a task matches a skill's description, the agent reads the full `SKILL.md` body into context.
3. **Execution**: the agent runs bundled scripts or reads `references/`/`assets/` files only as actually needed.

## 2. What agentskills.io specifies, and who maintains it

agentskills.io hosts the canonical specification, a client showcase, a validation reference library, and community links (GitHub, Discord). Per the site itself: "The Agent Skills format was originally developed by Anthropic, released as an open standard, and has been adopted by a growing number of agent products. The standard is open to contributions from the broader ecosystem."

- **Spec repo**: github.com/agentskills/agentskills (also mirrors a spec copy at github.com/anthropics/skills/blob/main/spec/agent-skills-spec.md).
- **Validator**: a `skills-ref` reference library/CLI (`skills-ref validate ./my-skill`) checks frontmatter validity and naming conventions.
- The spec deliberately covers only the file format and loading model — it does not standardize how an agent decides which skill to activate, sandboxing/permissions for scripts, or a package registry; those are left to each implementing agent. (Simon Willison's write-up on the launch — see Sources — independently describes the spec as "deliciously tiny," readable in minutes, with some aspects "deliberately underspecified.")
- Governance is Anthropic-originated but now open-contribution via the `agentskills` GitHub org; **[UNVERIFIED]** the exact legal/foundation structure (e.g., whether it sits under a neutral foundation vs. Anthropic-controlled repo) was not stated on the pages fetched.

## 3. Who supports the standard (verified against agentskills.io's own client list + vendor docs)

Confirmed via agentskills.io's official "Client Showcase" list (each entry links to that vendor's own skills documentation, cross-checked against the vendor's docs for several key ones):

- **Claude Code** and **Claude.ai** (Anthropic) — origin implementation; docs at code.claude.com/docs/en/skills and platform.claude.com/docs/en/agents-and-tools/agent-skills/overview.
- **ChatGPT & OpenAI Codex** — docs at developers.openai.com/codex/skills (redirects to learn.chatgpt.com/docs/build-skills). Confirmed via Simon Willison's post that OpenAI added Codex skills support the day after the standard's Dec 18, 2025 announcement.
- **GitHub Copilot** — docs.github.com/en/copilot/concepts/agents/about-agent-skills.
- **VS Code** — code.visualstudio.com/docs/copilot/customization/agent-skills.
- **Cursor** — cursor.com/docs/context/skills.
- **Gemini CLI** (Google) — geminicli.com/docs/cli/skills/.
- Also listed: JetBrains Junie, Goose (Block), OpenHands, opencode, Amp, Letta, Cursor, Roo Code, Kiro, Factory, Databricks Genie Code, Snowflake Cortex Code, Mistral Vibe, Spring AI, Laravel Boost, Tabnine, and roughly 20 more smaller/independent agent projects (full current list at https://agentskills.io/clients).

Partner-supplied skill packs at/around launch reportedly came from Box, Canva, Notion, Stripe, Zapier, Atlassian, and Figma (per secondary reporting — **[UNVERIFIED]**, not independently confirmed against each vendor's own page in this research pass).

**[UNVERIFIED]** Aggregate counts like "30+ platforms" or "20+ other platforms" appear consistently across multiple blog/press sources but the exact number moves depending on publish date since new adopters are added continuously; treat any specific count as approximate/time-of-writing.

## 4. Timeline

- **October 16, 2025** — Anthropic announces "Agent Skills" (claude.com/blog/skills, formerly anthropic.com/news/skills, redirects there): skills described as folders of instructions/scripts/resources Claude can load when needed, described as composable, portable, efficient, and powerful. Anthropic simultaneously open-sourced ~17 example skills at github.com/anthropics/skills. At this point Skills worked across Claude.ai, Claude Code, and the Claude API; Box, Canva, and Notion were named as early ecosystem partners integrating skills.
- **December 18, 2025** — Anthropic publishes Agent Skills as a vendor-neutral open standard, spinning the spec out to **agentskills.io** with a formal specification and validator, explicitly inviting any AI platform to adopt the format. (Corroborated independently by Simon Willison's Dec 19, 2025 blog post, which lists OpenCode, Cursor, Amp, Letta, Goose, GitHub, and VS Code as adopters named in the initial announcement.)
- **December 19–20, 2025** — OpenAI adds Skills support to Codex and joins the agentskills.io client showcase (per Willison's Dec 20 update note).
- **Since then (into 2026)** — continued adoption by Google Gemini CLI, GitHub Copilot, additional IDEs/agents, and enterprise platforms (Databricks, Snowflake, Spring AI, etc.), per the current agentskills.io client list. Exact adoption dates for each individual platform were **not** independently verified in this pass beyond OpenAI/Codex.

## 5. Skill marketplaces / plugin distribution

- Claude Code has a **plugin marketplace** system (code.claude.com/docs/en/plugin-marketplaces): a marketplace is a `marketplace.json` catalog listing installable plugins from git repos, local paths, npm packages, zip archives, or command output, giving centralized discovery, version tracking, and auto-updates.
- A **plugin** bundles multiple Claude Code extension types — skills, slash commands, sub-agents, hooks, MCP servers, LSP servers — as one installable unit; skills within a plugin live under `skills/<name>/SKILL.md` and are invoked namespaced as `/plugin-name:skill-name`.
- This plugin/marketplace layer is a **Claude Code-specific distribution mechanism**, distinct from (and layered on top of) the vendor-neutral agentskills.io file-format standard — the marketplace format itself is not part of the agentskills.io spec.
- Community-run marketplaces exist (e.g., third-party GitHub repos aggregating hundreds of plugins/skills), independent of Anthropic's official catalog. **[UNVERIFIED]** Scale claims such as "380,000+ monthly visitors" to marketplace directories come from a single secondary source (claudemarketplaces.com self-reporting) and were not independently corroborated.
- No equivalent "marketplace" concept is defined in the agentskills.io spec itself; other adopting agents (Codex, Cursor, etc.) each have their own (if any) distribution/discovery conventions for skills, which were not surveyed in depth here.

## Sources

- https://agentskills.io/specification
- https://agentskills.io (homepage, client showcase list)
- https://github.com/agentskills/agentskills
- https://github.com/anthropics/skills (Anthropic's official skills repo + spec mirror)
- https://claude.com/blog/skills (formerly anthropic.com/news/skills; Oct 16, 2025 announcement)
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- https://code.claude.com/docs/en/skills
- https://code.claude.com/docs/en/plugin-marketplaces
- https://simonwillison.net/2025/Dec/19/agent-skills/ (independent corroboration of Dec 18, 2025 open-standard launch and initial adopter list)
- https://developers.openai.com/codex/skills/ → https://learn.chatgpt.com/docs/build-skills
- https://docs.github.com/en/copilot/concepts/agents/about-agent-skills
- https://code.visualstudio.com/docs/copilot/customization/agent-skills
- https://cursor.com/docs/context/skills
- https://geminicli.com/docs/cli/skills/
- Secondary/press coverage consulted (not primary, used only for corroboration or flagged as unverified): unite.ai ("Anthropic Opens Agent Skills Standard"), venturebeat.com (Anthropic launch coverage), how2shout.com, dataconomy.com (OpenAI Codex skills), claudemarketplaces.com
