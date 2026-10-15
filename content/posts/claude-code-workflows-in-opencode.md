---
title: Taking My Claude Code Workflows to opencode
date: 2026-10-15
url: /blog/claude-code-workflows-in-opencode
author: Emilien Macchi
summary: How I run the same multi-agent workflow scripts in Claude Code and opencode, what it took, and how the two compare.
categories:
  - ai
  - code
---
A lot of my work goes through one skill: it plans a task, implements it in small commits, runs a
panel of reviewers and opens the PR. The interesting part is the workflow scripts: a few JavaScript
files that fan the work out to subagents, one reviewer per angle, then two skeptics per finding to
throw out the false positives.

Those scripts run on Claude Code's Workflow tool, so the most valuable part of my setup only worked
in one harness, with one vendor's models.

<!--more-->

### Why bother

Quota, mostly. A wide review can start 30+ agents, and on a busy week that eats a Max plan faster
than I'd like. I wanted a second harness for the days when saving quota matters more than speed:
[opencode](https://opencode.ai), pointed at GLM 5.3 behind an Anthropic-compatible API.

I also didn't want two copies to maintain: when I hand-copied my instructions to another agent, they
drifted. So the rule was: the scripts stay as they are, and opencode adapts.

### The easy part

opencode 2 reads `AGENTS.md` but not `CLAUDE.md`, so the shared rules moved to `AGENTS.md`:
`CLAUDE.md` imports it (`@AGENTS.md`) and opencode gets a symlink to it. Skills share the `SKILL.md`
format, and opencode reads `~/.claude/skills` directly. Agents need a small conversion script: add
`mode: subagent`, turn the tool allowlist into opencode denies, and drop the model so each harness
picks its own.

### The hard part: workflows

opencode has no Workflow tool. A workflow script is plain JavaScript that gets a few globals:
`agent()`, `parallel()`, `pipeline()`, `phase()`, `log()` and `args`. A simplified one:

```js
export const meta = { name: 'review', description: 'Review a diff, then try to refute each finding' }

const { findings } = await agent(`Review the changes in ${args.repo} since ${args.base}.`,
  { label: 'review', agentType: 'security-auditor', schema: FINDINGS })
const verdicts = await parallel(findings.map(f => () =>
  agent(`Try to refute this finding: ${JSON.stringify(f)}`, { label: `verify ${f.file}`, schema: VERDICT })))
return findings.filter((f, i) => verdicts[i]?.real)
```

So I wrote a Node runner that provides those globals and runs the scripts unchanged. The script
body, minus its `export`, becomes an async function, and each `agent()` call becomes its own
`opencode run` process:

```js
const script = new AsyncFunction('agent', 'parallel', 'pipeline', 'phase', 'log', 'args', body)

function runWorker(prompt, { agentType }) {
  const cli = ['run', '--standalone', '--format', 'json', '--auto']
  if (agentType) cli.push('--agent', agentType)
  const child = spawn('opencode', cli, { cwd: notesDir })
  child.stdin.end(prompt)
}
```

opencode's API couldn't enforce a JSON schema when I tried it (v2.0.28), so the runner appends the
schema to the prompt, takes the last valid JSON object from the reply, and asks again in the same
session if it doesn't validate. A failed agent returns `null` instead of throwing, so one flaky
worker doesn't take down a 30-agent review.

Workers never start in the repo. My own review panel flagged that as blocking on the first version:
a worker started in someone's PR checkout loads *their* opencode config and plugins. Now workers
start in a notes folder and reach the code by absolute path, and the runner hands them the repo's
`AGENTS.md` and `CLAUDE.md` read at the base commit, so a change can't rewrite its own reviewers'
rules. I run 8 workers at a time; 16 also worked, with a slow tail, at about 425 MB of RAM each.

The last piece is a thin opencode command that maps each Claude Code-only step:

```markdown
Load the `<your-skill>` skill and follow it with: $ARGUMENTS

- `Workflow({scriptPath, args})`: run `node run.mjs <script> <args.json> --out <file>`
  in the background, then read only the `--out` file.
- AskUserQuestion: use the `question` tool.
```

A test fails if the skill uses a Claude Code-only tool the command doesn't map, so they can't
quietly drift apart.

### How it compares

First, a spike: my unchanged review and brainstorming scripts on GLM 5.3. 35 of 40 agents returned
valid JSON on the first try, all 40 within one retry, and the review script caught all 4 bugs I had
planted in a test repo. At about a minute per agent, a 34-agent review took 9 minutes, 4 workers at
a time.

Then a less flattering test, on the runner's own PR. Claude Code's built-in `/code-review` on Opus
found 12 issues. My first opencode setup used a generic code-reviewer agent on GLM instead, and it
caught 1 of those 12. The same `/code-review`, run as headless Claude Code pinned to GLM, caught 5.
Same model, five times the catches: the harness and its review procedure mattered more than the
model. So opencode now runs those reviews through headless Claude Code on GLM.

|                     | Claude Code + Workflow  | opencode + my runner           |
|---------------------|-------------------------|--------------------------------|
| Models              | Built around Claude     | Any provider                   |
| Structured output   | Enforced by a tool call | Parsed from the reply, retried |
| Resume a failed run | `resumeFromRunId`       | Rerun the missing pieces       |
| Worker transcripts  | Kept                    | Deleted after the run          |
| Maintenance         | None                    | A ~760-line runner, plus tests |

My split today: Claude Code for big PRs, wide reviews and long sessions, opencode when quota is
tight. On GLM, I also want a second opinion from another model family before I trust an approval.

### What about plugins, or Pi?

My runner isn't an opencode [plugin](https://opencode.ai/docs/plugins/) (a module that adds hooks
and tools): it's a CLI the agent calls through bash. A plugin exposing a `workflow` tool could wrap
it, but opencode 2's plugin API was moving too fast to build on, and I didn't want 30-minute tool
calls inside its server. The [orchestration plugins](https://opencode.ai/docs/ecosystem#plugins) I
looked at, like subtask2 or oh-my-opencode, bring their own way to describe flows, so I'd be
rewriting my workflows.

[Pi](https://pi.dev) has [pi-dynamic-workflows](https://pi.dev/packages/pi-dynamic-workflows),
inspired by Claude Code's: same globals, same `export const meta`, and structured output through a
tool call instead of text parsing. The original is a prototype without resume, but [an extended
version](https://pi.dev/packages/@quintinshaw/pi-dynamic-workflows) adds `resumeFromRunId`,
`agentType`, per-agent models, worktree isolation and saved workflows. That's the piece I built by
hand, except native to the harness. I haven't tried it yet, so grain of salt, but if my scripts run
there with few changes, that's 760 fewer lines to maintain.

### Wrapping up

Keep your workflows harness-agnostic and push the harness-specific bits into a thin adapter you can
test. And benchmark on your own diffs: planted bugs are a low bar, and only a real PR showed my
first GLM reviewer missing 11 of 12 issues. My skill still has no opencode-specific line in it.
