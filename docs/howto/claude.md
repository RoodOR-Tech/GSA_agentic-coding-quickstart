---
title: "Building With Claude: A User Guide"
description: "What this repo is for, and what to expect, if you're using it to build something with Claude"
status: canonical
tier: 2
last_updated: "2026-09-24"
audience: "new users, developers"
keywords: ["claude", "claude code", "anthropic", "usai", "howto", "getting started"]
related_files: ["README.md", "docs/CONCEPTS.md", "docs/BACKEND_GUIDE.md", "docs/howto/acq.md", "docs/KNOWN_FAILURE_MODES.md"]
load_priority: "on-demand"
review_cycle: "quarterly"
---

# Building With Claude: A User Guide

> The [README](../../README.md) is the fastest path to a running sandbox for
> **any** supported agent. This guide is for one specific reader: someone who
> wants to build something **with Claude** using this repo, and wants to know
> why it exists and what to actually expect before they start.

---

## Why This Repo, If You're Building With Claude

Claude is capable of reading your files, writing code, and running commands on
your behalf. That's the point — it's also exactly what makes an unsupervised
agent dangerous if something goes wrong: a bad prompt, a malicious dependency,
or a compromised MCP server can turn "helpful coding agent" into "process with
your credentials and your filesystem." This repo exists to give Claude a box to
work in instead of your bare machine:

- **Isolation** — Claude runs inside a microVM (or container), not directly on
  your laptop. It cannot see your other projects, your host's full filesystem,
  or reach the open internet except through an allow-listed proxy.
- **Secret protection** — your USAi/GitHub/Anthropic credentials are injected
  into outgoing requests by the sandbox, not handed to Claude as plaintext
  environment values it could read and exfiltrate.
- **Federal behavioral rules, automatically loaded** — every sandbox pulls in
  the GSA [agentic-coding-playbook](https://github.com/GSA-TTS/agentic-coding-playbook)
  and links its `AGENTS.md` (least privilege, secure coding, prompt-injection
  defense, incident response, ...) and Agent Skills straight into Claude's own
  config paths. You don't write these rules yourself — Claude picks them up
  automatically the first time the sandbox starts.
- **A one-command bootstrap** — `acq run <agent> <path>` gets you all of the
  above without hand-rolling container flags, proxy config, or credential
  wiring yourself.

**What this is not:** a production or hosted environment, and not somewhere to
put PII or CUI. It's a *local development* sandbox — see the top of this
repo's `AGENTS.md` for the exact scope.

---

## Two Ways to "Build With Claude" Here — Pick One

This matters more than it looks, and picking wrong is the most likely way to
get stuck on your first run. `acq` supports several agents, and two of them
put you in front of Claude:

| | `acq run opencode .` | `acq run claude .` |
|---|---|---|
| What it runs | OpenCode, pre-configured to use Claude models | The actual **Claude Code** CLI |
| Model access | Routed through **USAi** (GSA's gateway) — works out of the box with the USAi key from Step 3 of the README | Needs its **own** Anthropic API key — USAi is not wired up for it (see below) |
| Default model | `claude-opus-5` (this is already the OpenCode kit's default — you're talking to Claude the moment it starts) | Whatever your Claude Code / Anthropic account gives you |
| Federal AGENTS.md + skills | Yes, linked automatically | Yes, linked automatically |
| Permission/approval policy | The sandbox's tuned OpenCode policy (default-allow, gated only on exfil-shaped actions — see below) | Claude Code's own normal approval prompts; nothing in this repo overrides them |
| Setup effort | None beyond the README's 5-Minute Quickstart | Extra manual steps — see below |

**If you just want to build something with Claude and don't care which
interface you use, start with `acq run opencode .`.** It's the fully wired,
zero-extra-setup path: the default model is already a Claude model, served
through the GSA-approved USAi gateway, with the federal rules and skills
already linked in.

**If you specifically want the Claude Code CLI** (its own UX, its own slash
commands, `CLAUDE.md`-driven memory, etc.), read on — it works, but it isn't a
one-command experience yet.

### Using the Claude Code CLI specifically

`acq run claude .` gives you the real Claude Code CLI, isolated the same way
as any other agent: same network proxy, same GitHub token handling, same
automatic `AGENTS.md`/skills linking (into `~/.claude/CLAUDE.md` and
`~/.claude/skills` inside the sandbox). Two things it does **not** get from
the default kit set:

1. **Model access.** The `usai-provider` kit that wires up USAi only
   configures OpenCode's own config file — it doesn't touch Claude Code's
   settings. You'll need to supply your own Anthropic API key:
   ```bash
   acq secret set anthropic --host api.anthropic.com --env ANTHROPIC_API_KEY
   ```
2. **Network egress.** The sandbox's network is deny-by-default with an
   allow-list, and `api.anthropic.com` isn't on it by default (only
   `api.gsa.usai.gov` and the GitHub hosts are). Without adding it, Claude
   Code's requests will simply hang or fail at the network layer even with a
   valid key. Adding a host to the allow-list means authoring a small extra
   kit (`caps.network.allow`) and passing it with `acq run claude . --kit
   <your-kit-ref>` — see `docs/BACKEND_GUIDE.md` and the
   `agentic-coding-playbook` kit's `spec.yaml` in the patterns repo for the
   shape of a kit spec.

This is a real gap in what ships today, not a hidden trick you're missing —
it's called out here so you don't spend an hour debugging a hang that's
actually a firewall. If you don't specifically need the Claude Code CLI's own
interface, `acq run opencode .` sidesteps both issues entirely.

---

## What to Expect, Step by Step

Assuming the `opencode` path (see above for why): follow the README's
[5-Minute Quickstart](../../README.md#5-minute-quickstart) — install `acq`,
then run:

```bash
acq run opencode .
```

1. **First run takes a minute or two.** `acq` boots a microVM, installs
   OpenCode, and fetches the federal playbook and USAi config kits. You'll see
   progress as it goes. Later runs against the same folder are fast.
2. **You'll be prompted for a USAi key** (create one at the
   [USAi key console](https://gsa.usai.gov/console/key-management); keys
   expire every 7 days) **and, if your project has a GitHub repo, offered a
   GitHub token.** You can decline the GitHub prompt and add one later.
3. **You land in OpenCode, already talking to Claude.** The default model is
   `usai/claude-opus-5`. Nothing else to configure.
4. **Claude already knows the federal rules.** `AGENTS.md` (least privilege,
   secure coding, data handling, prompt-injection defense, ...) and the
   playbook's Agent Skills are already linked into the sandbox — Claude reads
   them the same way it would any project's own instructions file.
5. **Permission prompts are sandbox-tuned, not silent.** Ordinary work (read,
   edit, most bash) is allowed without a prompt — the sandbox itself is the
   security boundary. You'll only be asked to approve actions that open a
   **new outbound destination**: `git push`, adding a git remote, `gh pr
   create`, `scp`/`rsync`/`nc`, or a data-uploading `curl`/`wget`. That's by
   design (see `docs/BACKEND_GUIDE.md`), not a bug if it feels more permissive
   than you expected.

---

## Verify the Federal Rules Actually Loaded

The playbook fetch is **non-fatal by design**: if it fails (offline, an
expired GitHub token, a bad ref), the sandbox still starts — just without the
rules and skills, and with only a warning printed during startup that's easy
to miss. Don't assume alignment happened; check it:

```bash
acq ls                                                  # find your sandbox's name
acq exec <name> -- test -f ~/.agentic-coding-playbook/AGENTS.md && echo "playbook OK"
```

For the Claude Code CLI specifically, also confirm the link landed in its own
config path:

```bash
acq exec <name> -- ls -la ~/.claude/CLAUDE.md ~/.claude/skills
```

If either check fails, re-run `acq run <agent> <path>` (re-attach heals a
missing kit) and confirm your GitHub token is still valid — see
[docs/KNOWN_FAILURE_MODES.md](../KNOWN_FAILURE_MODES.md) if it still doesn't
land.

---

## What This Repo Deliberately Doesn't Promise

- **Not a hard behavioral guarantee.** The playbook's `AGENTS.md` and skills
  are advisory context Claude is instructed to follow, not a technical control
  — the actual security boundary is the sandbox (network allow-list, no host
  filesystem access, injected-not-exposed secrets). Don't rely on the rules
  alone to stop a determined or badly-prompted agent from doing something
  unwanted inside the sandbox.
- **The rules clone is writable**, by design, so Claude *could* edit its own
  `CLAUDE.md`/skills mid-session. Blast radius is one disposable sandbox; a
  fresh `acq run` re-fetches a clean copy.
- **Not a production or ATO'd environment.** No PII, no CUI, local development
  only — see this repo's own `AGENTS.md`.

---

## Next Steps

- **Set up your actual project properly** (repo hygiene, ADRs, test
  discipline): the [agentic-coding-playbook](https://github.com/GSA-TTS/agentic-coding-playbook) —
  the same rules your sandboxed Claude is already following.
- **Understand what's under the hood** (kits, customization, optional
  integrations): [docs/CONCEPTS.md](../CONCEPTS.md)
- **Choose/compare sandbox backends** (msb vs sbx):
  [docs/BACKEND_GUIDE.md](../BACKEND_GUIDE.md)
- **Something not working?** [docs/KNOWN_FAILURE_MODES.md](../KNOWN_FAILURE_MODES.md)
