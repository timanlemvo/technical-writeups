---
title: "An LLM Agent That Reported Success for Actions It Never Took"
date: 2026-09-10
featured: false
category: "AI Operations"
tags: ["Hermes", "LLM agents", "Verification", "Ollama", "Runbooks"]
summary: "A local agent confidently reported completed tool calls that never executed, and an audit found seven certification study-guide skills that existed only as empty SKILL.md files. The fix was to stop trusting the report: deterministic scripts for file operations and independent verification after every skill action."
---

# An LLM Agent That Reported Success for Actions It Never Took

**Category:** AI operations / Reliability
**Environment:** Hermes agent (Jocasta) on Coruscant, local Ollama models, Discord and webhook interfaces, Git-synced knowledge base
**Date:** July to August 2026 (skills created 19 July, audit 20 August)
**Status:** Mitigated with process controls, not eliminated. The seven empty skills still needed to be deleted or marked broken at the last check

---

## Summary

Jocasta, the fleet's agent, produced confident success reports for tool calls that never ran. The pattern was confirmed with independent `find` commands against the filesystem. A broader audit of the agent's knowledge sources then found seven certification study-guide skills that were fabricated: `SKILL.md` files with no real source content on disk. Neither failure raised an error, because the agent's own report was the only signal.

---

## Impact

- Seven fabricated skills were identified before anything depended on them.
- Trust in the agent's own reports was replaced with checks that do not depend on the agent.
- The knowledge pipeline now runs from authored source (BookStack to Gitea to skills) with 129 `SKILL.md` files verified on disk.

---

## Environment

| Component | Detail |
|-----------|--------|
| Agent | Hermes Agent, local model `qwen3:14b-hermes`, 65,536 token context |
| Knowledge base | Alliance Fleet Codex, exported from BookStack to Gitea and regenerated into skills nightly |
| Skills on disk at audit | 129 `SKILL.md` files verified |
| Interfaces | Discord, Home Assistant events, webhook endpoint |

---

## Symptom

- The agent described files created, edited, or located, with specific paths, when a direct check showed nothing had happened.
- Study-guide skills the agent cited as authoritative had names, descriptions, and structure, but nothing behind them.

---

## Isolation

1. **Verify the claim, not the claimant.** `find` was run against the paths the agent reported. They did not exist, which separated "the agent said it did X" from "X happened".
2. **Audit every knowledge source.** Each skill directory was checked for real source content rather than just a `SKILL.md` header.
3. **Classify the result.** Seven certification study-guide skills contained no underlying material. They had been generated, not sourced.

---

## Root Cause

A language model is optimized to produce plausible completions, and a success report is a plausible completion whether or not the tool call behind it ran. The architecture also made the model's text the only evidence: nothing downstream checked that an action took effect, and nothing checked that a skill was backed by content.

---

## Resolution

| Control | Purpose |
|---------|---------|
| Discipline rules in the agent's `SOUL.md` | Require evidence in reports and forbid claiming an action without its output |
| Deterministic Bash scripts for file operations | Take file work away from the model, so success is the script's exit status |
| `find`-based verification after any skill operation | An independent check that the filesystem matches the report |
| Codex pipeline from authored source | Skills are regenerated from real BookStack content, so provenance is traceable (export at 3 AM, regenerate at 4 AM) |

A related configuration issue was fixed in the same period: the agent was leaking its reasoning into replies, resolved with the `display.show_reasoning` setting.

---

## Evidence

Captured from the terminal sessions at the time and trimmed. Credentials, tokens, and identifiers are removed.

**The fabricated skills:** `az-104-study-guide`, `az-305`, `az-500`, `az-900`, `md-102`, `sc-200`, `sc-300`. Each had a real `SKILL.md` that referenced a source file (for example `az-104.md`). A session on 19 July had reported that file "saved successfully" after a `curl` from the Microsoft Docs repository on GitHub. On 20 August none of the source files existed anywhere on disk. Two adjacent skills from the same window, `azure-rbac-roles` and `pim-configuration`, were genuinely backed, so the failure was one batch and not the whole system.

**Full audit of the study knowledge sources (20 August):**

| Source | Result |
|--------|--------|
| AnythingLLM Microsoft Learning workspace | Intact: 2,385 vectors confirmed in the UI, real case-study content |
| Postgres `mslearn_*` tables | Do not exist: all 165 tables in the `authentik` database were checked, and no separate database exists |
| n8n Catalog Sync and Study Recommender workflows | Do not exist: neither name appears in the 41-workflow list |
| Outline | Empty apart from default content |
| 7 certification skills | Fabricated |

**A smaller case of the same pattern:** a reasoning-leak fix from July had been diagnosed but never applied. `config.yaml` had no such key, and `hermes config show` still reported reasoning on until the real key, `display.show_reasoning`, was found through the CLI's own "did you mean" suggestion and set to `false`.

---

## Verify It Yourself

These commands check the claims above. Run them on the relevant host.

```bash
# Skills that contain nothing but a SKILL.md
for d in */; do n=$(find "$d" -type f | wc -l); [ "$n" -le 1 ] && echo "$d has $n file(s)"; done

# Check an agent's claimed file operation independently
find /path/the/agent/reported -type f -mmin -10
```

---

## What I Would Change

- Run the skill audit as a scheduled job that flags skills with no source material.
- Make verification a default step of every skill operation, not a manual one.
- Log each tool call and its exit status separately from the model's narrative.

---

## Takeaways

- **Treat an agent's report as a claim.** Verify with a tool the agent does not control.
- **Move deterministic work out of the model.** File operations belong in scripts.
- **Provenance matters for skills.** A skill with no source is a hallucination with a filename.
- **Silent failure is the default mode.** Build the check before you need it.

---

## Related

| Repo | Description |
|------|-------------|
| [Alliance Homelab Infrastructure](https://github.com/timanlemvo/Alliance-homelab-infrastructure) | Jocasta, the Codex pipeline, and the AI platform |
