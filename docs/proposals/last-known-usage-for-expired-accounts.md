# Design proposal: show last-known windows and reset times for expired accounts

Status: proposal (no code changes in this PR)

## Problem

When an account enters a sentinel state, `cswap list` replaces its whole usage
block with the sentinel note and a single summary line:

```
  1: alice@example.com [Work]
     re-login needed — refresh token dead; log in with Claude Code, then run: cswap add
     └ last seen 40% used · 1d ago

  2: bob@example.com [Work]
     re-login needed — refresh token dead; log in with Claude Code, then run: cswap add
     └ last seen 99% used · 3d ago

  3: carol@example.com [Personal] (active)
     ├ 5h:     29%   resets Sep 21 00:10  in 1h 22m
     ├ 7d:     87%   resets Sep 22 01:00  in 1d 2h
     └ Fable:  77%   resets Sep 22 01:00  in 1d 2h
```

Accounts 1 and 2 are the ones the operator most needs to plan around, and they
are the two that say the least. Two questions go unanswered:

1. **Which window is at 99%?** `last_seen_note` reports
   `100 - account_headroom(last_good)`, which is the *binding* window only. A
   5h window at 99% and a 7d window at 99% mean completely different things:
   one clears in hours, the other in days.
2. **When does it clear?** No reset time at all. The operator cannot tell
   whether re-logging in to account 2 buys them a usable account or a maxed
   one.

The information exists. `switcher._usage_entry_lines` discards it:

```python
if entry.sentinel is not None:
    out = [dimmed(SENTINEL_NOTES.get(entry.sentinel, entry.sentinel))]
    last_seen = last_seen_note(entry)
    if last_seen is not None and entry.sentinel != USAGE_API_KEY:
        out.append(f"{dimmed('└')} {muted(last_seen)}")
    return out
```

`entry.last_good` is the full normalized usage dict, per-window percentages and
`resets_at` included. The sentinel branch collapses it to one number before the
formatter that knows how to render windows ever sees it.

## The reset times are still correct

This is the part worth being precise about, because it is what makes the fix
worth doing rather than just cosmetic.

`resets_at` is an absolute UTC instant, and `oauth.fresh_reset_strings`
already recomputes the countdown and the clock string at render time for
exactly this reason — its docstring says the fetch-time strings drift as the
measurement ages, so it never uses them when `resets_at` is present.

A reset instant does not decay with the measurement that carried it. A 7-day
window observed three days ago with `resets_at` four days out still resets four
days out. So "when does this account clear" is answerable today for every
expired account that has a stored measurement. `cswap` simply declines to
answer it.

The percentages are the part that ages, and they age in a known direction. An
account cswap cannot authenticate cannot spend through cswap, so its measured
usage is a floor, not an estimate: it can only have grown if the operator used
that account somewhere else. Two consequences:

- A window whose `resets_at` has **already passed** is known to have rolled.
  Reporting its three-day-old 99% as the current state, which is what account 2
  does now, is not merely stale, it is backwards. The most likely truth is 0%.
- A window whose `resets_at` is **still ahead** keeps its measured percentage as
  a lower bound on usage, which is the conservative direction for a
  switch-planning decision.

## Proposal

Render the stored measurement through the normal window formatter, and classify
each window at render time by comparing its `resets_at` to now.

### Rendering

```
  1: alice@example.com [Work]
     re-login needed — refresh token dead; log in with Claude Code, then run: cswap add
     ├ last known · 1d ago
     ├ 5h:      —    rolled over since
     ├ 7d:     40%   resets Sep 22 01:00  in 1d 2h
     └ Fable:  31%   resets Sep 22 01:00  in 1d 2h

  2: bob@example.com [Work]
     re-login needed — refresh token dead; log in with Claude Code, then run: cswap add
     ├ last known · 3d ago
     ├ 5h:      —    rolled over since
     ├ 7d:     99%   resets Sep 22 01:00  in 1d 2h
     └ Fable:  99%   resets Sep 22 01:00  in 1d 2h
```

Three per-window states:

| State | Condition | Rendering |
| --- | --- | --- |
| Carried | `resets_at` in the future | measured percentage, plus countdown and clock recomputed by `fresh_reset_strings` |
| Rolled | `resets_at` in the past | `—` and `rolled over since`; no percentage is asserted |
| Unknown | no `resets_at` stored | measured percentage, no reset column |

The `last known · <age>` header carries the provenance once for the whole
block, so no individual line has to repeat it, and the reader is never invited
to mistake the block for a live reading.

Rolled windows deliberately print `—` rather than `0%`. Zero would be an
assertion cswap cannot make: the operator may have used that account directly in
Claude Code. `—` says the stored number expired without replacing it with a
guess.

### Behavior notes

- **No new configuration.** This replaces one line with a block of the same
  shape every other account already prints. Operators who want the terse form
  have `cswap list --json`.
- **`USAGE_API_KEY` keeps its current behavior.** An API-key account has no
  quota windows, and the existing code already suppresses the last-seen line for
  it.
- **No stored measurement, as today.** Accounts with `last_good is None` still
  print the sentinel note alone.
- **Ordering stays as-is.** This proposal does not change `account_headroom`,
  autoswitch, or any ranking. It is a display change only. Whether a rolled
  window should also stop binding `account_headroom` is a real question, and a
  separate one.

### Surfaces to keep in parity

`SENTINEL_NOTES` and `last_seen_note` are shared deliberately so the surfaces
stay word-for-word identical. The same applies here:

- `switcher._usage_entry_lines` — the CLI block above.
- `tui/widgets.py:200` and `tui/data.py` — the dashboard, which calls
  `last_seen_note` today.
- `menubar.py:399` — has less vertical room; the minimum is the binding
  *carried* window rather than the binding window overall, so the menu stops
  showing a percentage that has since rolled.
- `json_output.last_good_usage_fields` — already emits the full stored dict and
  its age, so consumers can classify themselves. Worth confirming `resets_at`
  survives into the JSON payload unmodified rather than adding a derived field.

## Implementation sketch

1. Add a window classifier in `oauth`, next to `fresh_reset_strings`, returning
   carried / rolled / unknown for a window dict and a reference time. Injecting
   `now` keeps it testable without freezing the clock.
2. Give `_format_usage_lines` a mode that renders rolled windows as `—`, so the
   sentinel branch and the normal branch share one formatter.
3. Replace the sentinel branch's `last_seen_note` call with the header line plus
   the formatted block.
4. Point the TUI and the menu bar at the same helpers.
5. Tests: a stored measurement whose 5h window has rolled but whose 7d has not;
   all windows rolled; a measurement with no `resets_at`; an API-key account;
   and a sentinel account with no measurement at all.

`last_seen_note` stays exported until the TUI and the menu bar have moved over,
then goes.

## Alternatives considered

**Leave it and tell the operator to re-login.** Re-logging in is the action the
proposal helps them decide *between*, when several accounts are expired. Account
2 at 99% on a 7-day window is not worth re-logging in to today; account 1 at 40%
is. Today both look the same.

**Show the full block with no rolled/carried distinction.** Simpler, but it
reprints account 2's three-day-old 99% on a 5h window as though it were current.
That is the specific misreading this proposal exists to remove.

**Assert 0% for rolled windows.** Tempting, and usually right, but cswap cannot
see usage spent outside it. `—` is the honest symbol.
