# Monitoring GitHub Changes Safely

For agents tracking repo changes autonomously:

## Pattern 1: Bounded Observation
Monitor only ONE repo at a time. Record which one. Never drift.

## Pattern 2: Change Detection
Poll every 5 minutes (not constantly):
- git fetch
- git log origin/main..HEAD
- Record changes with timestamp

## Pattern 3: Action Gating
Only act on changes if:
- Change is new (not seen before)
- Actor is known (verify commit author)
- Content is reviewed (don't auto-act on untrusted changes)

## Pattern 4: State Recording
Record: what changed, when, who made it, what we did in response.

## Pattern 5: Alert System
If unexpected changes detected: log and notify, don't auto-fix.

**Enables safe autonomous monitoring across multiple repos.**
