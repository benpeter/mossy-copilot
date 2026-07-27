# One harness, any driver at any level

Written 2026-07-27 as an evaluation. **Decided by the Farmer 2026-07-28 00:10:
do it.** Fold `mossy-copilot` back into `mossy-bottom`, make the driver a per-role
choice, and support running every role on Copilot. The live royalairmaroc
migration is the guinea pig: it cuts over to the merged harness and proves it.

The fork was the right way to find out what differs. It is the wrong way to keep
it.

The reason is not tidiness. It is that the fork has already started costing
correctness. Two of tonight's six escalations were harness defects that existed
ONLY because the fork forked: a window-sizing bug that has nothing to do with
Copilot but is fixed in one repo and not the other, and a timmy regression I
introduced in the fork while upstream's timmy stayed correct. Every hour the two
diverge, the chance grows that a fix lands on the wrong side.

## What the fork actually changed, measured

Four files. That is the whole delta, and it is why the merge is cheap.

| file | change | Copilot-specific? |
|---|---|---|
| `bin/barn.sh` | `role_cmd`, `ready_pattern`, `trust_pattern`, `trust_keys`, per-role `launch_cmd`, a copilot entry in `preflight_tools` | the seam is generic, the values are not |
| `bin/barn.sh` | `window_size` / `size_window` | **no**, purely generic |
| `timmy/bin/timmy` | `has_worker_spinner`, two markers in `isboxchrome`/`ischrome`, `●` alongside `⏺` | additive, both dialects live side by side |
| `prompts/shaun.md` | a worker-dialect section | Copilot-specific prose |

Note what is NOT in that list: the deference chain, the trust rule, the evidence
rule, `send-verified.sh`, `heartbeat.sh`, `stuck-check.sh`, `rotate.sh`,
`bitzer.md`. The architecture was driver-agnostic already. That is the finding.

## Why the merge is easy now and was not before

Because timmy already speaks both dialects. That was the hard part and it is done.

timmy's core is a double-snapshot diff, which never cared what was running in the
pane. Everything above it is cue detection, and cues are additive: `has_spinner`
for Claude Code, `has_worker_spinner` for Copilot, ORed in `classify_once`. A
third driver adds a third function and one more `||`. Nothing has to be
parameterised, nothing has to be selected at runtime, and a pane running either
driver classifies correctly with no configuration at all.

**The classifier does not need to know who is driving.** That is the property that
makes one harness work, and it is worth protecting: resist any future design that
passes a `--driver` flag into timmy. If a cue needs to know which driver it is
looking at, the cue is not anchored tightly enough.

## The shape to merge into

One repo, `mossy-bottom`. A driver becomes a small table rather than a fork:

    DRIVERS[claude]=cmd:"claude --model opus --dangerously-skip-permissions"
                    ready:"bypass permissions on"
                    trust:"trust this folder"        keys:"Enter"
    DRIVERS[copilot]=cmd:"copilot --model gpt-5.6-sol --effort xhigh --allow-all"
                     ready:"/ commands"
                     trust:"Confirm folder trust"    keys:"Down Enter"

and a per-role selector, defaulting every role to claude so the current behaviour
is byte-identical:

    MOSSY_DRIVER_SHIRLEY=copilot bin/barn.sh up <target>

That reuses the exact seam `MOSSY_INJECT_<ROLE>` already established, so it is a
pattern the harness has rather than a new concept. `role_cmd`, `ready_pattern`,
`trust_pattern` and `trust_keys` collapse into table lookups. Everything else
stays.

The prompt problem is the only fiddly bit: `prompts/shaun.md` needs a
worker-dialect section whose content depends on which driver the worker runs.
Split it: `prompts/shaun.md` stays driver-agnostic, and
`prompts/workers/<driver>.md` carries the dialect. barn concatenates the right one
into the boot prompt. That is strictly better than the fork's approach, which
inlines Copilot facts into a file that also has to be right for Claude Code.

## Order of work

1. Backport `window_size`/`size_window` to mossy-bottom first, on its own. It is
   generic, it is already tested, and it fixes a real bug there today: a detached
   window is 80 columns, the three-way split gives each role 26, both TUIs wrap
   their footer, and boot_pane then reports healthy panes as failed.
2. Port the timmy Copilot cues. Additive, no behaviour change for Claude Code, and
   the existing 60-test suite is the regression guard.
3. Introduce the driver table with every role defaulting to claude. Prove
   byte-identical `--plan` output before and after.
4. Split the worker dialect out of `shaun.md`.
5. Archive `mossy-copilot` with a pointer.

Steps 1 and 2 are worth doing even if the rest never happens, because they are
bug fixes wearing a refactor's clothes.

## The open question the merge does not answer

Whether the drivers should be able to differ at the TOP of the chain, not just the
bottom. Tonight says the value of the chain came from shaun, not shirley: all six
escalations came from the driver watching the worker. If that holds, the
interesting configuration is a cheap worker and an expensive driver, which is what
ran tonight. But nobody has tried a cheap DRIVER, and that is the experiment the
merged harness would make trivial and the fork makes tedious. One more argument
for merging.

## Addendum: running the DRIVERS on Copilot too

**Corrected 2026-07-28. The paragraph below was wrong.** It was read off a model
list rather than probed. Probing directly with `copilot -p OK --model <name>`,
which fails instantly and free on an unavailable model:

| model | available |
|---|---|
| `claude-opus-4.8` | yes |
| `claude-sonnet-5` | yes |
| `claude-sonnet-4.5` | yes |
| `gpt-5.6-sol` | yes |
| `claude-opus-5` | **no** |

So "run Opus through Copilot" means 4.8, not 5, and the available set is gated by
org policy rather than by the CLI. That is the more useful fact: a harness that
hardcodes a model name breaks for somebody else's org, which is why the merged
harness takes `MOSSY_MODEL_<ROLE>` as its own override.

The current version of this document is `docs/drivers.md` in mossy-bottom.

~~Checked 2026-07-27 against Copilot CLI 1.0.75. Its model list includes
`claude-opus-5`, `Claude Opus 4.8/4.7/4.6/4.5`, `Claude Sonnet 5`, `Claude Haiku
4.5` alongside the GPT and Gemini families. So "run Opus through Copilot" is
available, and the whole chain could sit on one plan.~~

What it would take, in order of effort:

1. **timmy: nothing.** Already classifies Copilot panes. This was the hard part
   and it is done. It is what makes the rest cheap.
2. **barn: the driver table from the section above**, roughly 40 lines, with the
   per-role selector defaulting to claude so nothing changes unless asked.
3. **The prompts: the actual work.** `shaun.md` and `bitzer.md` assume Claude Code
   in ways that are load-bearing, not cosmetic:
   - The compaction cadence sends `/compact <keep-string>`. Copilot's `/compact`
     takes no focus string, so the driver has to re-anchor by hand afterwards.
     The fork already documents this for the worker; it would have to apply to
     the drivers themselves, including shaun compacting shaun.
   - Context management keys on the `Context: N%` meter. Copilot shows AI credits
     instead, so the STANDBY-at-threshold rule needs a different trigger, most
     likely slice boundaries only.
   - bitzer wakes a compacted shaun. That flow assumes the driver can be
     compacted and resumed the way Claude Code does it.
4. **Unmeasured: cost.** Nobody has measured what `claude-opus-5` costs in AI
   credits per driver-hour. Tonight's drivers ran 5 to 7 minute xhigh turns
   continuously, so the answer is not obviously cheaper than a Claude
   subscription; it just moves which meter runs.

The honest summary: the mechanism is a day's work and mostly prompt surgery, the
enabling piece is already built, and the reason to do it is billing consolidation
or model comparison rather than capability. Worth doing AFTER the merge in the
section above, not instead of it.
