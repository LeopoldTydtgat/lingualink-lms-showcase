# ADR 4: Backup and disaster recovery

**Status:** Adopted, in production - with openly tracked gaps

**Date:** 2026

## Context

The database is the business: schedules, balances, class history, pay records. Losing it would not be an inconvenience, it would be an extinction event for the platform. Threats worth planning for: provider failure, account compromise, and - statistically most likely - operator error, a bad migration or a destructive statement against the live database.

The platform runs on a managed provider (paid tier), operated by one person, with no budget for enterprise DR tooling.

## Decision

Layered protection, cheapest and most probable risks first.

- **Managed daily backups** on the database provider's paid tier are the baseline recovery mechanism.
- **Process controls against operator error**, because the most likely disaster is self-inflicted: destructive DDL is only run after verifying the live production code no longer references the column or table being dropped (checked against the actual deployed branch, not memory); DDL runs one statement at a time; and every schema change is captured as a timestamped migration file afterwards so the schema's history is reconstructable.
- **Test accounts are strictly separated from real data**, and any write against live rows is preceded by a read confirming exactly whose data it touches.
- **Offsite copy, independent of the provider account:** planned and tracked - a scheduled database dump to storage outside the provider, so a compromised or closed provider account does not take the backups with it.

## Alternatives considered

- **Provider backups only.** Simplest, but backups living inside the same account as the database share its fate. Rejected as the end state, accepted as the baseline.
- **Continuous replication to a second provider.** Strongest protection, meaningful cost and complexity. Rejected as disproportionate for the platform's size; the daily-backup plus offsite-dump model loses at most a day, which the business can survive.

## Consequences

**Good**
- The most probable disaster (operator error) is defended by process at zero cost.
- The baseline backup requires no maintenance to keep working.

**Costs and sharp edges - stated honestly**
- **A backup you have never restored is a hypothesis.** The restore path has not yet been exercised end to end, and until it is, recovery time and completeness are assumptions. This is tracked as an open item, not filed as done.
- The offsite copy is likewise still an open item. Until it exists, provider-account compromise remains the weakest scenario.
- Daily backups mean up to a day of data loss in the worst case. Accepted deliberately for this scale.

## What I would tell someone building this

Rank your disasters by probability, not by drama. Provider meltdown makes a better story, but the statement you ran against the wrong table is the disaster you will actually meet - and process, not tooling, is the defense. Then be honest about state: a backup strategy has exactly two verified facts, "a restore worked when I tested it" and everything else. Track the gap in writing rather than rounding it up to done.
