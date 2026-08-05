---
name: monitoring-github-changes
description: Poll GitHub issue/PR/notification state for changes worth acting on, without re-alerting on your own just-completed actions or missing a genuine terminal-state signal
scope: any agent running a background watch loop over GitHub issues, PRs, or account-wide notifications across one or more repos
trigger: about to arm a polling loop against GitHub state, or reviewing why an existing one is either too noisy or has gone silent
---

# Monitoring GitHub Changes

## The problem this solves

A naive GitHub-state poll loop (diff current vs. previous snapshot, emit a
line on any difference) has two opposite failure modes that both look like
"it's working":

1. **Self-echo** — the loop emits an event confirming an action you just
   took yourself. You already know you created the PR; being told again
   wastes the monitor's actual purpose (flagging things you don't already
   know) and trains you to skim past its output.
2. **Silent on the case that matters** — a filter written for the happy
   path (e.g. "emit on merge") says nothing if the thing crashes, gets
   closed unmerged, or just never resolves. Silence then looks identical
   to "still fine," which is the worst possible failure mode for a
   monitor — the whole point was to replace "assume it's fine" with
   verified knowledge.

## Core discipline

1. **Suppress self-authored creation events; never suppress state changes.**
   A `NEW PR`/`NEW issue` event where you are the author is something you
   already know — filter it by author before emitting. A `STATE CHANGE`
   event (merged, closed, review submitted) is never safe to suppress by
   author, because someone *else* acting on your own artifact is exactly
   the useful case — confirmed live: a fire-alarm loop that filtered NEW
   events by author but left STATE CHANGE events unfiltered correctly
   surfaced a real review landing on a self-authored PR, the same tick a
   duplicate self-echo of the PR's own creation was correctly silenced.
2. **Enumerate every terminal state your filter should catch, not just the
   success path.** Before arming a poll loop, ask: if this crashed/got
   rejected/stalled right now, would my filter emit anything? If the
   answer for any terminal state is no, widen the filter — a monitor that
   only greps for the success marker is indistinguishable from a dead
   monitor during exactly the periods you most need it alive.
3. **Diff against a persisted snapshot, not a fixed baseline.** Re-seed the
   snapshot to current state after each real dispatch action you take
   yourself (a PR you open, an issue you file) — otherwise your own next
   poll cycle treats your own artifact as "new" and re-triggers the
   self-echo problem discipline 1 exists to prevent.
4. **A single account-wide notification stream and a single repo-scoped
   fire-alarm are different tools, not redundant ones.** The former
   (`gh api notifications`) catches @-mentions, review requests, and
   assignments across every repo you have access to — but is blind to
   plain state transitions on repos nobody explicitly notified you about.
   The latter (a targeted `gh pr list`/`gh issue list` diff loop against
   one repo) catches those transitions but only for the repo it's
   pointed at. Run both if your actual responsibility spans more than
   one repo and includes both "things aimed at me" and "things happening
   in repos I steward."
5. **For genuinely org-wide coverage, `gh search` across explicit
   `--owner` flags beats iterating every repo individually.** Two calls
   (`gh search issues --owner OrgA --owner OrgB ...`, same for `prs`) can
   cover dozens of repos in one pass; iterating repo-by-repo doesn't scale
   and burns rate limit for no real benefit. Accept the search API's
   eventual-consistency lag as a real tradeoff of this approach — relax
   any "did this shrink unexpectedly" guard rather than treating normal
   indexing delay as a bug.

## Minimal reference shape

A poll loop with both disciplines applied:

```bash
while true; do
  current=$(gh pr list --repo OWNER/REPO --state all --json number,title,state,author,updatedAt --limit 100)
  node -e '
    const fs = require("fs");
    const current = JSON.parse(process.argv[1]);
    const stateFile = process.argv[2];
    const selfAccount = process.argv[3];
    const prev = JSON.parse(fs.readFileSync(stateFile, "utf8"));
    const next = {};
    for (const pr of current) {
      const key = String(pr.number);
      next[key] = pr.state;
      const prevState = prev[key];
      if (prevState === undefined) {
        if (pr.author.login !== selfAccount) {
          console.log(`NEW PR #${pr.number} [${pr.state}]: ${pr.title} (author: ${pr.author.login})`);
        }
        // self-authored creation events recorded silently, never emitted
      } else if (prevState !== pr.state) {
        // state changes ALWAYS emit, regardless of author
        console.log(`STATE CHANGE PR #${pr.number}: ${prevState} -> ${pr.state} (${pr.title})`);
      }
    }
    fs.writeFileSync(stateFile, JSON.stringify(next));
  ' "$current" "$STATE_FILE" "$SELF_ACCOUNT"
  sleep 300
done
```

## Explicitly out of scope for this skill

- **Distributed caching across multiple independent agent processes** — a
  real, harder problem (cache invalidation, consistency, avoiding N agents
  all polling the same repo independently) that this skill's disciplines
  don't yet solve. If your actual need is a shared cache multiple agents
  read from rather than N independent poll loops, that's a genuinely
  bigger design than what's documented here — don't assume this skill
  covers it just because the org's registered description for this repo
  once claimed it did.
- **Which specific card/thread should act on a surfaced event** — that's a
  routing decision (see `RULES-OF/agent-tagging`), not a monitoring one.
  This skill's job ends at "here is a verified, de-duplicated event";
  what happens next is a separate concern.

## Provenance

Extracted 2026-08-04/05 from real, live-operated monitor loops (an
account-wide notification watcher and a single-repo fire-alarm, both
iterated through several versions the same night after real false
positives and one real correction from the Org Lead: "should a fire alarm
confirm your own work?"). This repo previously existed with a
description claiming this capability but zero actual content — filed and
fixed as `SKILL-OF/monitoring-github-changes#1`.
