# Krun Agent Skills

Official [Agent Skills](https://agentskills.io) for building with [Krun](https://krun.ai), the decision API that
turns text, images, documents and audio into typed answers with probabilities.

The skill teaches coding agents to design Krun integrations: when Krun fits and when it doesn't, which primitive to
use, how to act on uncertainty, how to batch judgments for latency, and where to find the current API and SDK details
in the [live docs](https://docs.krun.ai/llms.txt).

Read it here: [`skills/krun/SKILL.md`](skills/krun/SKILL.md).

## Install

### Claude Code

```bash
claude plugin marketplace add krun-ai/skills
claude plugin install krun@krun-ai
```

### Other agents (skills.sh)

```bash
npx skills add krun-ai/skills --skill krun
```

The CLI asks which agents to install to. Installs are project-local by default; add `-g` for a global install.

### Manually

Copy `skills/krun/` into your agent's skills directory, for example `.claude/skills/krun/`.

## Use

Ask your agent for the outcome you want, for example:

> Use Krun to route incoming support tickets and send uncertain decisions to human review.

> Use Krun to classify uploaded invoices and flag uncertain documents for review.

In Claude Code you can also invoke the skill directly with `/krun:krun`.

| Skill | What it covers |
|---|---|
| [`krun`](skills/krun/SKILL.md) | Decision design, primitive selection (`choice`, `noul`, `score`, `multi`), uncertainty and escalation, multimodal context, latency, and pointers to the current docs |

Calling the Krun API requires an API key (`KRUN_API_KEY`). Krun is in closed beta: [request access](https://krun.ai).

## License

[Apache-2.0](LICENSE).
