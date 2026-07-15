<!--
  Managed by claude-utils (svnscha/claude-plugins).
  This file is the canonical, versioned copy of my personal ~/.claude/CLAUDE.md.
  Edit it here in the repo — not per device — then sync any device with:
      /claude-utils:onboard
  so every machine stays consistent.
-->

# Personal instructions

General working style adapted from the Andrej Karpathy skills guidelines
(github.com/multica-ai/andrej-karpathy-skills). These bias toward caution over
speed; for trivial tasks, use judgment.

## Think before coding
Don't assume. Don't hide confusion. Surface tradeoffs.
- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop, name what's confusing, and ask.

## Simplicity first
Minimum code that solves the problem. Nothing speculative.
- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or configurability that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it. If a senior engineer
  would call it overcomplicated, simplify.

## Surgical changes
Touch only what you must. Clean up only your own mess.
- Don't "improve" adjacent code, comments, or formatting, and don't refactor
  what isn't broken.
- Match existing style, even if you'd do it differently.
- Remove imports, variables, and functions that your changes made unused; leave
  pre-existing dead code alone — mention it, don't delete it.
- Every changed line should trace directly to the request.

## Goal-driven execution
Define success criteria, then loop until verified.
- Turn tasks into verifiable goals: "add validation" → "write tests for invalid
  inputs, then make them pass"; "fix the bug" → "write a test that reproduces it,
  then make it pass".
- For multi-step tasks, state a brief plan with a verify check per step.
- Strong success criteria let you work independently; weak ones ("make it work")
  force constant clarification.

## Git commits
- Never add a `Co-Authored-By: Claude ...` trailer, a `<noreply@anthropic.com>`
  trailer, or any "Generated with Claude Code" line to commit messages or PR
  bodies. Write commit messages as if I authored them.

## READMEs & docs
- Write READMEs for the reader who wants to use the thing: lead with a neat
  overview of what's on offer, then how to install and use it. Leave out
  repository-layout diagrams, internal-structure walkthroughs, and other
  maintainer/plumbing sections.
