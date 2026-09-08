# ADR 5: Timezone handling across three portals

**Status:** Adopted, in production

**Date:** 2026

## Context

Teachers and students are in different countries. A class is one moment in time, but every participant must see it on their own clock: the teacher in her timezone, the student in his, the admin in hers. Reminder emails, "join" buttons that activate minutes before class, billing periods, and calendar sync all hang off the same times.

Timezone bugs are uniquely dangerous here because they look plausible. A class shown an hour off does not crash anything - someone simply misses a lesson, which costs real money and real trust.

## Decision

One rule at the core: **store one truth, render per viewer.**

- Every class time is stored once, as an absolute moment (UTC) in the database. Stored times are never pre-converted for anyone's locale.
- Each user profile carries an explicit timezone, and every display of a time converts from the stored absolute moment to the viewer's profile timezone at render time.
- Time math (reminder scheduling, join-window activation, "is this within 24 hours") is done on absolute moments, never on wall-clock strings.
- A small set of banned patterns is enforced by convention, because each caused or nearly caused real bugs:
  - No `toISOString()` to build local dates - it silently shifts the date across the UTC boundary for evening or morning users.
  - No locale-formatting functions in code that renders on both server and browser - the server and the viewer disagree, and the page breaks on load.
  - Calendar components are pinned to an explicit timezone mode rather than trusting defaults.
- External calendar sync converts at the boundary in both directions, so a busy-block created in one calendar lands at the same absolute moment in the other.

## Alternatives considered

- **Store times in the business's local timezone.** Intuitive for the admin, wrong for everyone else, and breaks twice a year at daylight-saving transitions. Rejected.
- **Store the wall-clock time plus a timezone label per row.** Preserves intent but makes every comparison and every reminder calculation a conversion problem. Rejected: absolute moments compare and sort natively.
- **Let the browser guess the user's timezone.** Convenient until someone travels or uses a VPN, and their whole schedule silently shifts. Rejected in favour of an explicit profile setting the user controls.

## Consequences

**Good**
- One class, one stored truth, any number of correct views. New surfaces (emails, exports, sync) reuse the same conversion rule.
- Daylight-saving transitions are handled by conversion libraries at render time instead of by hand-rolled offset math.

**Costs and sharp edges**
- Discipline is the cost: a single `toISOString()` in the wrong place reintroduces the off-by-a-day class of bug, so the banned list is enforced in review, every time.
- Every user must have a timezone set, which makes it a required field, not an optional preference.
- Testing requires deliberately crossing boundaries - viewer behind UTC, viewer ahead of UTC, class at midnight, class on a daylight-saving switch day - because the happy path hides these bugs completely.

## What I would tell someone building this

Decide where conversion happens and allow it nowhere else. Times are stored absolute, compared absolute, and become local only at the last moment before a human sees them. Then test with a user whose evening is another user's next morning - if your date grouping, reminders, and join windows survive that pair, the model is sound.
