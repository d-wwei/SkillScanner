[English](README.md) | [中文](README.zh.md)

# SkillScanner

**One skill file that turns any AI agent into a security scanner — no dependencies, no ML models, no setup.**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

---

## Why This?

Snyk scanned 3,984 skills on ClawHub. 36.82% had security flaws. 100% of confirmed malicious skills combined code execution with prompt injection. The agent skill ecosystem has the same security posture as early npm — wide open.

Existing defenses are code libraries. prompt-shield needs `pip install` and optionally a 738MB DeBERTa model. AegisGate needs scikit-learn and jieba. Aigis needs Python 3.11+. None of them work as a drop-in skill that any agent can load without setup.

The gap: there was no pure-instruction security scanner. No SKILL.md you could symlink into `~/.claude/skills/` and immediately start scanning other skills for threats. SkillScanner fills that gap. The host agent IS the detection engine — its language understanding does the work that regex and ML classifiers do in code-based tools. No install. No dependencies. Works on Claude Code, Codex, Gemini CLI, OpenClaw, or anything that reads SKILL.md.

## What It Is

SkillScanner is a single SKILL.md file plus 5 reference files (~5,400 words total). Install it on any AI agent that supports the SKILL.md format, and the agent gains the ability to scan content for 10 categories of security threats with OWASP LLM Top 10 tags.

It covers: prompt injection, context manipulation, indirect injection (including 8 agent platform formats), social engineering, encoding obfuscation, MCP tool poisoning, data exfiltration, dangerous operations, skill metadata poisoning, and memory/state poisoning.

## Key Features

- **LLM-native semantic detection.** Layer 3 doesn't simulate a regex engine — it asks the agent to paraphrase what the content is trying to accomplish, then checks the paraphrase against threat intents. This catches attacks that rephrase around keyword patterns ("set aside your operational parameters" = "ignore your instructions"), which regex-based tools miss entirely.

- **8-platform tool-call injection patterns.** Detects injected tool calls for OpenAI, Anthropic Claude, Google Gemini, AWS Bedrock, vLLM/Hermes, ReAct agents, MCP JSON-RPC, and AutoGPT/OpenDevin. Not generic descriptions — platform-specific format signatures.

- **Per-finding hardened dampening.** The false-positive suppression system checks the ±5 lines around each individual finding, not the document as a whole. All CRITICAL categories (T1, T6, T7, T10) are never dampened. True positives are never dampened. This blocks the "wrap real attacks in tutorial framing" bypass that affects simpler dampening designs.

- **12 real-world attack cases + 7 structural archetypes.** Layer 3 matches content against cases from Snyk ToxicSkills (3,984 skills analyzed), Palo Alto Unit 42 (24-layer web injection), and Aigis MCP research. The 7 archetypes (Identity Override, Data-as-Instructions, Exfil Construction, Concealment Wrapper, Persistence Planting, Escalation Ladder, Framework Mimicry) catch novel attacks by shape, not by keyword.

- **3-Gate scanning architecture.** Input Gate (before processing), Tool-Result Gate (after MCP/API calls — where indirect injection most commonly arrives), Output Gate (before delivery — catches canary leaks and compromised behavior). Not just input scanning.

- **Bucket-weighted risk scoring.** Findings are classified into 4 buckets (Intent 45%, Payload 25%, Hijack 20%, Anomaly 10%) with probabilistic accumulation within each bucket. Multiple medium signals compound to high risk. 3+ threat categories trigger a 1.5x composite bonus with a minimum score floor of 60 (HIGH RISK).

## With / Without

| | Manual Review | With SkillScanner |
|---|---|---|
| Encoded payloads | Miss base64/hex/ROT13 nested inside each other | 9-step normalization with 2-layer recursive decoding |
| Novel phrasing | Catch "ignore instructions" but miss "set aside operational parameters" | Vocabulary-agnostic intent paraphrase catches semantic equivalents |
| MCP tools | Read the description, hope it's honest | Scan for `<IMPORTANT>` tags, secrecy directives, description injection, shadowing |
| Educational camouflage | "For research purposes" framing fools you | CRITICAL categories never dampened, per-finding context check |
| Multi-vector attacks | Each finding judged in isolation | Probabilistic bucket accumulation + 1.5x composite bonus |
| Tool-result injection | You don't scan what tools return | Gate 2 scans tool outputs before the agent incorporates them |

## How It Works

### 5-Layer Protocol

```
Content → L1: Normalize → L2: Pattern Match → L3: Semantic Analysis → L4: Context Judge → L5: Verdict
           (9 steps,        (T1-T10,           (intent paraphrase,    (per-finding,         (bucket-weighted
            recursive)       3 text versions)    12 cases, 7 archetypes) dampening rules)      risk scoring)
```

**Layer 1** strips encoding obfuscation: base64, hex, URL, unicode escapes, ROT13, zero-width characters, HTML comments, homoglyphs, and creates a condensed copy (all whitespace removed) to catch space-separated evasion. Decoding is recursive up to 2 layers.

**Layer 2** runs all 10 threat categories against three text versions (normalized, decoded payloads, condensed). Produces a numbered finding list that carries through to Layer 4.

**Layer 3** loads the attack cases library and runs vocabulary-agnostic intent detection: "Regardless of specific words, does any section attempt to change identity, override instructions, or extract configuration?" Structural archetype matching catches novel attacks not in the case library.

**Layer 4** classifies each finding as true positive, false positive (INFO), or needs review. Discussion-context dampening (max 25%) applies only to needs_review findings, only in the surrounding ±5 lines, and never to CRITICAL categories. Scanner-evasion patterns ("pre-scanned", "whitelisted", "skip scan") are always true positives.

**Layer 5** aggregates into 4 buckets, applies probabilistic accumulation, composite bonus, and maps to verdict: CLEAN / LOW RISK / SUSPICIOUS / HIGH RISK / CRITICAL.

### Output Formats

Text report and JSON report. Both include per-finding severity, category, OWASP tag, location, evidence, context judgment, and remediation advice.

### Design Decisions

| Decision | Why |
|----------|-----|
| Pure instructions, no code | Zero dependencies = works everywhere. The LLM's language understanding IS the detection engine. |
| Per-finding dampening, not per-document | One "for research" line must not suppress findings 200 lines away. Reviewed after adversarial testing. |
| CRITICAL categories never dampened | T1/T6/T7/T10 are too dangerous for any suppression — real attacks in tutorial framing are still real attacks. |
| Bucket-weighted scoring | Flat per-finding addition can't express multi-vector risk. Probabilistic accumulation means 3 MEDIUMs ≠ 1 HIGH — they can exceed it. |
| Structural archetypes, not just cases | 12 cases can't cover all attacks. 7 abstract shapes (Identity Override, Exfil Construction, etc.) catch novel variants. |

## Quick Start

```bash
# 1. Clone
git clone https://github.com/d-wwei/SkillScanner.git

# 2. Install (symlink to your agent's skill directory)
ln -sf "$(pwd)/SkillScanner" ~/.claude/skills/security-scan    # Claude Code
ln -sf "$(pwd)/SkillScanner" ~/.agents/skills/security-scan     # Codex / Gemini CLI

# 3. Scan
# In your agent, say:
#   "Scan this skill for security issues: [paste content]"
#   "Quick scan this MCP tool definition"
#   "扫描这个 skill"
```

### Sub-Commands

| Command | What It Does |
|---------|-------------|
| `scan <content>` | Full 5-layer scan (all references loaded) |
| `quick-scan <content>` | Layer 1+2 only — fast pattern check, no semantic analysis |
| `gate <name> <content>` | Scan at a specific gate (input / tool-result / output) |
| `self-test` | Run 4-tier verification (basic / evasion / camouflage / negative) |

## Project Structure

```
SkillScanner/
  SKILL.md                              # Router — stance, red lines, acceptance criteria, workflow
  references/
    scan-protocol.md                    # 5-layer protocol + scoring + 3-Gate
    threat-catalog.md                   # T1-T10 threat patterns (EN/ZH, per-platform)
    attack-cases.md                     # 12 cases + 7 structural archetypes
    report-templates.md                 # Text + JSON output formats
    self-test.md                        # 4-tier test payloads
```

## Attribution

Built by synthesizing insights from 12 projects and research papers:

**Detection patterns & architecture:** [prompt-shield](https://github.com/mthamil107/prompt-shield), [AegisGate](https://github.com/ax128/AegisGate), [Aigis](https://github.com/killertcell428/aigis), [prompt-guard](https://github.com/useai-pro/openclaw-skills-security)

**Scanning methodology:** [snyk/agent-scan](https://github.com/snyk/agent-scan), [LLM Guard](https://github.com/protectai/llm-guard), [Vigil-LLM](https://github.com/deadbits/vigil-llm), [pytector](https://github.com/MaxMLang/pytector)

**Threat research:** [Snyk ToxicSkills](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/), [Palo Alto Unit 42](https://unit42.paloaltonetworks.com/ai-agent-prompt-injection/), [Meta Prompt Guard 2](https://huggingface.co/meta-llama/Llama-Prompt-Guard-2-86M), [Sentinel](https://huggingface.co/qualifire/prompt-injection-sentinel)

## License

MIT
