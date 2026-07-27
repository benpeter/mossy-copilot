# Porting the chain to a Copilot worker

Everything learned putting GitHub Copilot CLI in the worker seat on 2026-07-27,
during the first live run (a royalairmaroc.com migration to AEM Edge Delivery).
Written so the next person does not rediscover any of it.

The shape: **the worker runs Copilot, both drivers stay on Claude Code.** That is
a deliberate one-variable ablation. `prompts/shaun.md` and `prompts/bitzer.md` are
600 lines written for Claude Code, and swapping all three panes at once would have
told us nothing about Copilot specifically.

## What Copilot is, mechanically

Version 1.0.75. Launched as
`copilot --model gpt-5.6-sol --effort xhigh --allow-all`.

- `--allow-all` is the equivalent of `--dangerously-skip-permissions`. Nobody is at
  the keyboard to approve a tool call.
- **Leave `--no-ask-user` OFF.** Answering the worker's questions is what shaun is
  for; disabling the ask tool removes the thing the chain exists to do.
- `--effort` takes `none|minimal|low|medium|high|xhigh|max`. Settings persist in
  `~/.copilot/settings.json`, so the flags are belt and braces.
- It has `/compact`, `/context`, `/clear`, `/usage`, `/skills`. `/compact` takes no
  focus string, unlike Claude Code, so a driver sends a bare `/compact` and then
  re-anchors the worker itself.
- **A slash command needs TWO Enters.** Typing `/` opens a filter palette; the
  first Enter picks the highlighted entry out of it and the second submits. One
  Enter leaves the command sitting in the composer, which reads exactly like a send
  that silently failed.

## The TUI, which is what the harness actually reads

Copilot keeps its composer box rendered for the whole turn and changes only the
FOOTER. Claude Code puts its spinner ABOVE the box. That single difference caused
two separate false-idle defects.

| state | footer |
|---|---|
| idle | `/ commands · ? help · tab next tab` + model name right-aligned |
| busy | `◉ Working · 7.2 KiB esc interrupt` |
| busy, labelled | `◉ Committing failing header test · 8.9 KiB esc interrupt` |
| status line above the box | `<cwd> [⎇ branch]` + `Session: N AIC used` |

Three traps, all of which bit:

1. **The busy cue is below the box, not above it.** `has_spinner` looks upward from
   the fence and finds nothing, so the pane keeps the idle-box shape and a
   busy-but-quiet worker reads idle. `has_worker_spinner` reads BELOW the fence.
2. **The word `Working` is not reliable.** Copilot swaps the current task name in
   where it would be. Key on the interrupt affordance instead. That is safe only
   because the match is position-anchored below the last rule-fenced box, so a
   transcript quoting the phrase cannot fire it, and there is a fixture asserting
   exactly that.
3. **Never fall back to motion.** Before fix 2, the verdict came from
   `sustained_motion`, which was decided by whether the two snapshots happened to
   land on the same animation frame of an alternating `◉`/`◎` glyph. Same worker
   state, two different answers. A coin flip wearing the clothes of a measurement,
   and worse than a consistent bug because it hides.

Copilot marks an assistant turn with `●` where Claude Code uses `⏺`. Match those as
**two anchored alternatives, never a bracket class**: awk matches a bracket
expression byte-wise, so `[⏺●]` also matches every other U+2xxx glyph in the pane,
including the box rules, and the turn marker lands on chrome.

## Skills

Copilot reads `~/.copilot/skills/`, which on this machine is a symlink to
`~/.claude/skills/`. Two things follow, and both cost time:

- **Copilot registers a skill by the `name:` in its frontmatter, not by the
  directory it sits in.** Symlinking a skill under a prefixed directory name does
  not rename it. Adobe's EDS skills ship as a Claude Code plugin under
  `~/.claude/plugins/cache/`, invisible to Copilot; symlinking them in works, but
  they resolve as `building-blocks`, `da-content` and so on, never as a prefixed
  variant.
- **Copilot loads its skill set once, at session start.** A skill added afterwards
  is invisible to that session and the only fix is a respawn through
  `barn.sh relaunch shirley`.

Those two defects masked each other for two hours: the session predated the
symlinks AND the names would not have resolved anyway.

## Git

Copilot adds `Co-authored-by: Copilot <...>` to commits it makes. There is no
config knob; it constructs the trailer in the commit command. If the project bans
tool attribution, say so in the guardrails explicitly and check the branch BEFORE
the first push, because removing it afterwards is a history rewrite.

## Cost

Measured, not estimated, at `gpt-5.6-sol` + `xhigh`:

| work | AI credits |
|---|---|
| trivial one-line reply | 9.3 |
| small shell task | 14.5 |
| reading a 157-line guardrails file | 19.5 |
| a full measure-and-report slice | 60 to 120 |

Roughly 400 credits/hour in steady state. The counter is in the pane footer, so a
driver can record it every tick and the run's cost is visible without leaving tmux.

## What was NOT a problem

Worth stating, because the interesting result is how little had to change:

- The deference chain, the trust rule and the evidence rule needed no changes.
- `send-keys` delivery works identically; `send-verified.sh` needed nothing.
- timmy's core double-snapshot detector is TUI-agnostic and was already right.
- Copilot held strict TDD across six commits without being reminded, and produced
  a header matching the live site to the pixel on every measured box.
- Every blocker it hit, it reported instead of working around: it refused to patch
  the measurement tool, refused to rewrite history, and refused to hand back a
  measurement it could not take.
