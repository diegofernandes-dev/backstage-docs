# Agent Prompts

This directory stores the canonical long-form prompts used to execute tightly scoped architecture and implementation checkpoints.

The intent is to avoid repeatedly pasting large prompts into an agent session. A short launcher prompt may simply instruct the agent to fetch `backstage-docs@main`, read the relevant prompt in this directory, verify the current architecture/docs baseline, and execute it exactly.

## Rules

- Always `git fetch origin main` before using a prompt.
- The current canonical architecture/docs take precedence over assumptions embedded in an older prompt.
- A prompt authorizes only the work it explicitly marks as GO.
- STOP gates are mandatory.
- Agents must update canonical evidence/docs only with factual results.
- Do not expand scope to "helpfully" implement the next checkpoint.

## Current prompts

- [`d1-kargo-fit-evaluation.md`](./d1-kargo-fit-evaluation.md) — execute the narrow D1 promotion-controller fit evaluation and decide `KARGO_FIT` vs `KARGO_NO_FIT` while preserving the carried D0 production-hardening gaps.
