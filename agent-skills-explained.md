# Agent Skills — A Clean-Cut Explanation

## What a Skill is

A Skill is a folder containing instructions that teach Claude how to handle a specific task or workflow. You write the guidance once; Claude loads it automatically whenever a request matches.

Think of it as an onboarding guide for a new hire, not a plugin. It carries procedural knowledge — workflows, house style, domain rules — plus optional scripts and reference material.

Skills differ from the neighbouring concepts:

| Thing | What it gives Claude |
|---|---|
| Prompt | One-off instructions for this conversation |
| MCP server | Access to tools and live data ("what Claude *can* do") |
| Skill | Workflow knowledge ("how Claude *should* do it") |

MCP is the kitchen; a Skill is the recipe.

## Progressive disclosure — the core mechanic

Skills are filesystem objects, and Claude reads them with bash on demand. Content loads in levels:

| Level | Content | When loaded | Cost |
|---|---|---|---|
| 1 | YAML frontmatter (`name`, `description`) | Always, at startup | ~100 tokens per Skill |
| 2 | SKILL.md body | When the Skill triggers | Under ~5k tokens |
| 3 | `references/`, `assets/` | When Claude reads them | Zero until accessed |
| 3 | `scripts/` | Run via bash | Only the output enters context |

Two consequences matter. Installing many Skills costs almost nothing until one triggers. And bundled content is effectively unbounded — a Skill can ship a 200-page API reference that costs zero tokens on every request that doesn't need it.

Scripts are the sharpest tool here: the code never enters context, only its output, and execution is deterministic where language interpretation is not.

## File structure

```
your-skill-name/
├── SKILL.md          # required — instructions + YAML frontmatter
├── scripts/          # optional — executable code
│   └── validate.py
├── references/       # optional — docs Claude reads as needed
│   └── api-guide.md
└── assets/           # optional — templates, fonts, icons used in output
    └── report-template.md
```

## Hard rules

- The file must be named exactly `SKILL.md`. Case-sensitive, no variations.
- The folder name must be kebab-case: `notion-project-setup`. No spaces, underscores, or capitals.
- No `README.md` inside the Skill folder. Documentation goes in SKILL.md or `references/`. A repo-level README for human visitors is fine and separate.
- `name`: max 64 chars, lowercase letters/numbers/hyphens only. Cannot contain "claude" or "anthropic" (reserved).
- `description`: non-empty, max 1024 chars.
- No XML angle brackets (`<` `>`) anywhere in the frontmatter — it lands in the system prompt, so this is an injection guard.

## Frontmatter

Minimum:

```yaml
---
name: your-skill-name
description: What it does. Use when the user asks to [specific phrases].
---
```

Optional fields:

```yaml
license: MIT
compatibility: Requires Python 3.11 and network access   # 1–500 chars
allowed-tools: "Bash(python:*) Bash(npm:*) WebFetch"
metadata:
  author: Company Name
  version: 1.0.0
  mcp-server: server-name
```

### The description does all the routing work

It is the only thing Claude sees before deciding to load the Skill. Structure it as **what it does + when to use it + trigger phrases**.

Good:

```yaml
description: Analyzes Figma design files and generates developer handoff
  documentation. Use when the user uploads .fig files, asks for "design specs",
  "component documentation", or "design-to-code handoff".
```

Bad:

```yaml
description: Helps with projects.                          # too vague
description: Creates multi-page documentation systems.     # no triggers
description: Implements the Project entity model.          # no user language
```

Mention file types when relevant. Include the words users would actually say.

## Writing the body

Recommended shape:

```markdown
# Skill Name

## Instructions
### Step 1: [First major step]
What happens, and the exact command:
`python scripts/fetch_data.py --project-id PROJECT_ID`
Expected output: [what success looks like]

## Examples
User says: "Set up a new marketing campaign"
Actions: 1. Fetch existing campaigns  2. Create with parameters
Result: Campaign created with confirmation link

## Troubleshooting
Error: [message] → Cause: [why] → Solution: [fix]
```

Rules that hold up in practice:

- Be specific and runnable. "Run `python scripts/validate.py --input {file}`" beats "validate the data before proceeding."
- Put critical instructions at the top. Buried rules get skipped.
- Replace ambiguous language with explicit checks. "CRITICAL: before calling `create_project`, verify name is non-empty, one team member assigned, start date not in the past."
- For validations that must not fail, bundle a script instead of trusting prose.
- Keep SKILL.md under ~5,000 words. Push everything else into `references/` and link it.
- Skills compose — assume other Skills are loaded alongside yours.

## Where Skills live

| Surface | How to install | Sharing scope |
|---|---|---|
| Claude Code | `~/.claude/skills/` (personal) or `.claude/skills/` (project) | Personal, project, or via plugins |
| claude.ai | Settings → Capabilities → Skills, upload a zip | Per user; admins can deploy org-wide |
| Claude API | `/v1/skills` endpoints, referenced in `container.skills` | Workspace-wide |

Skills do not sync across surfaces. Upload separately to each.

Runtime differs too. Claude Code gives full network access and normal filesystem reach. The API sandbox has no network access and no runtime package installation — only preinstalled packages — and requires the code execution tool.

## Testing

Three things to test, in order:

1. **Triggering.** Run 10–20 queries. It should fire on obvious asks and paraphrases, and stay quiet on unrelated ones. Aim for ~90% on relevant queries.
2. **Function.** Correct outputs, successful tool calls, error paths, edge cases.
3. **Baseline comparison.** Same task with and without the Skill. Count messages, tool calls, failures, tokens.

Debugging trick: ask Claude "when would you use the [skill-name] skill?" It quotes the description back, which shows you exactly what's missing.

Fastest path to a good Skill: iterate on one hard task in a normal conversation until Claude gets it right, then extract that winning approach into a Skill. Expand to more cases afterward.

## Failure modes and fixes

| Symptom | Fix |
|---|---|
| Never triggers | Description too generic. Add trigger phrases, technical keywords, file types. |
| Triggers on everything | Narrow the scope; add negative triggers ("Do NOT use for X — use the Y skill instead"). |
| Loads but instructions ignored | Too verbose, buried, or ambiguous. Trim, move critical rules to the top, replace prose checks with scripts. |
| Slow or degraded responses | SKILL.md too large, or too many Skills enabled. Move detail to `references/`; audit if you have 20–50+ enabled. |
| Won't upload | `SKILL.md` misspelled, missing `---` delimiters, unclosed quotes, or a name with spaces/capitals. |

## Common Skill shapes

1. **Document and asset creation** — embedded style guides, templates, quality checklists. No external tools needed.
2. **Workflow automation** — step-by-step processes with validation gates and refinement loops.
3. **MCP enhancement** — sequences multiple MCP calls, embeds domain expertise, handles common API errors.

Useful patterns within those: sequential orchestration with explicit dependencies and rollback; multi-MCP coordination in phases; iterative refinement against a quality script; context-aware tool selection via a decision tree; domain rules applied before action (compliance, safety, audit trail).

## Security

Install Skills only from sources you trust. A Skill directs Claude to run code and invoke tools, so a malicious one can exfiltrate data or misuse system access.

Audit every bundled file — SKILL.md, scripts, assets — for network calls or file access that doesn't match the stated purpose. Skills that fetch external URLs carry extra risk, since fetched content can carry instructions and remote dependencies can change after you vetted them. Treat installing a Skill like installing software.

## Checklist

**Before you start:** 2–3 concrete use cases identified · tools identified · folder structure planned.

**During development:** kebab-case folder · `SKILL.md` spelled exactly · `---` delimiters present · `name` kebab-case · `description` covers what *and* when · no `<` `>` · instructions actionable · error handling included · examples provided · references linked.

**Before shipping:** triggers on obvious asks · triggers on paraphrases · stays quiet on unrelated asks · functional tests pass · tool integration works.

**After shipping:** watch for under/over-triggering · collect feedback · iterate the description first · bump `metadata.version`.

## Sources

- [Agent Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [anthropics/skills](https://github.com/anthropics/skills)
- [The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)
