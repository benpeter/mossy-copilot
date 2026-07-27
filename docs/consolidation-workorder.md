# Work order: fold the fork, make the driver a per-role choice

Farmer's decision, 2026-07-28 00:10. Read `one-harness.md` first for the why and
the measured delta; this file is the how, in order, with the risks named.

**Where the work happens:** `~/github/benpeter/mossy-bottom`, on a branch. NOT on
its main, because that repo is driving the live contitires run and a `relaunch`
there would pick up whatever is on disk.

**The guinea pig:** the live royalairmaroc run in tmux `ramsvc:ram`. It currently
boots from `~/github/benpeter/mossy-copilot` (its panes carry
`MOSSY_REPO_DIR=/Users/ben/github/benpeter/mossy-copilot`). It cuts over to the
merged harness only after the harness is green, and only at a slice boundary.

## Order

**1. Backport `window_size` / `size_window`.** Generic, already tested, fixes a
real bug in mossy-bottom today: a detached window is 80 columns, the three-way
split gives each role 26, both TUIs wrap their footer, and `boot_pane` then
reports healthy panes as failed. Cherry-pick `8693139` (red) and `da1cef4`
(green), plus the Q section pinning `window-size manual` against a client resize.

**2. Port the timmy Copilot cues.** Additive: `has_worker_spinner`, the two
markers in `isboxchrome`/`ischrome`, `●` alongside `⏺` as TWO ANCHORED
ALTERNATIVES never a bracket class. mossy-bottom's existing 60-test suite is the
regression guard, and the fork's four Copilot fixtures come along.

**3. The driver table.** Replace `role_cmd`, `ready_pattern`, `trust_pattern`,
`trust_keys` with a table keyed by driver name, and a per-role selector
`MOSSY_DRIVER_<ROLE>` reusing the `MOSSY_INJECT_<ROLE>` seam. **Every role
defaults to claude**, and the proof of that is byte-identical `bin/barn.sh up
--plan` output before and after. Add `MOSSY_MODEL_<ROLE>` so a role can pick a
model without redefining the whole command.

**4. Split the worker dialect out of `shaun.md`.** `prompts/shaun.md` becomes
driver-agnostic; `prompts/workers/<driver>.md` carries the dialect; barn
concatenates the right one into the boot prompt. This is strictly better than the
fork, which inlines Copilot facts into a file that must also be right for Claude
Code.

**5. Make the DRIVERS runnable on Copilot.** This is the new work, not a port.
Three things in the prompts are load-bearing Claude Code assumptions:
   - `/compact <keep-string>`. Copilot's `/compact` takes no focus string, so a
     Copilot-driven shaun must re-anchor by hand after compacting, and a
     Copilot-driven bitzer must do the same when it compacts shaun.
   - The `Context: N%` meter. Copilot shows `Session: N AIC used` instead, so the
     STANDBY-at-threshold rule needs a different trigger. Use slice boundaries as
     the primary and drop the percentage backstop rather than invent a proxy.
   - A slash command needs TWO Enters on Copilot (the palette eats the first).
     `send-verified.sh` must learn this or every driver-issued slash command
     silently fails to submit.

   Model choice for the drivers: `claude-opus-5` is available through Copilot, so
   driver quality need not drop. Cost per driver-hour is UNMEASURED; measure it on
   the guinea pig before concluding anything about economics.

**6. Cut the RAM run over.** At a slice boundary, with the worker idle:
   - relaunch bitzer and shaun first, verify they boot and re-anchor
   - then relaunch shirley
   - all three with `MOSSY_REPO_DIR` pointing at mossy-bottom
   A relaunch is a hard context loss for that pane, so it is worth exactly one
   boundary. bitzer and shaun rehydrate from MISSION, GUARDRAILS, TICKS and
   CHRONICLE by design; shirley gets re-anchored by shaun as after any respawn.

**7. Archive `mossy-copilot`** with a pointer to mossy-bottom, once the RAM run
has produced an accepted slice on the merged harness. Not before: the fork is the
rollback.

## Risks, and what to do about each

- **mossy-bottom is live.** Work on a branch. Never merge to its main while the
  contitires chain is running unless the Farmer says so.
- **The cutover loses driver context.** Do it at a boundary, bitzer and shaun
  first so a failure is caught before the worker is disturbed.
- **A wrong driver table silently boots a role-less pane.** `boot_pane` fails
  quietly by design: it waits out its timeout and warns. After the cutover, check
  every pane reached its input box before trusting the run.
- **Two Enters.** Until `send-verified.sh` handles it, any slash command a
  Copilot-driven driver sends will look delivered and do nothing.

## Done means

`bin/barn.sh up --plan` byte-identical for an all-claude chain; both suites green
in mossy-bottom; the RAM chain running on the merged harness with all three roles
on Copilot; and one slice accepted end to end after the cutover.

## Open design question the Farmer raised: `/new` instead of `/compact`?

Asked 2026-07-28 00:15: should shirley and shaun get a FRESH session at each
boundary rather than a compaction?

**For the worker: yes, and it is probably strictly better.** The argument is that
the re-anchor cost is already being paid. shaun's compaction procedure is
`/compact` and then restate who she is, the mission line, the binding guardrails
and the current slice. If he restates all of that anyway, the only thing
compaction preserves is the residue of the slice just finished, which the
compaction focus-string explicitly tries to DROP. So compaction buys a lossy
summary of exactly what we want gone, plus whatever wrong beliefs the summary
carries forward. A fresh session gives a deterministic starting state instead.

Evidence from tonight: shirley was respawned once mid-run, for the skills defect.
shaun re-anchored her and she continued without a stumble, and her next slice
matched live to the pixel. That is one datapoint in favour.

Cost: re-reading GUARDRAILS costs about 20 AI credits, against 60 to 120 for a
slice. Cheap. On Copilot the in-pane `/clear` ("abandon this session and start
fresh") is the mechanism, no respawn needed; a full `barn.sh relaunch` is the
heavier fallback and is what you need if the skill set changed.

**For the driver: no, keep compaction and STANDBY.** shaun's accumulated read on
the worker IS the product. Tonight's best diagnosis, that the verdict was being
decided by animation phase, came from him holding three samples taken minutes
apart and noticing they disagreed. A shaun who starts fresh each slice cannot
connect observations across slices, and connecting them is the whole reason the
driver layer exists. He already rehydrates from MISSION, GUARDRAILS, TICKS and
CHRONICLE, so the machinery is there; the risk is dropping the unwritten
judgment, and the unwritten judgment is the value.

**Test it on the guinea pig.** Run the worker on fresh-session-per-slice and the
driver on compaction, and record in TICKS whether worker quality changes. That is
one flag, `MOSSY_BOUNDARY_<ROLE>=compact|fresh`, and it is a natural companion to
the driver table in step 3.
