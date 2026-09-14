# Operating notes

A dated log of real production incidents and decisions from building and running this platform. Each entry covers what happened, what I changed, and the resulting improvement. Newest first. All entries are retyped from the private repository's history and sanitised.

**14 Sep 2026 - Invoice status ground truth**
The database check constraint allowed the statuses submitted and late, but older code and documentation claimed uploaded and overdue, values that never existed. I corrected the code to match the database, and built an admin-only unmark-paid that safely reverses a mistaken payment marking and recomputes state. The database is now treated as the single source of truth, and admin mistakes are recoverable without touching data by hand.

**25 Aug 2026 - Booking money-path audit**
A retried or double-clicked booking could in theory deduct hours twice, and a lost database response could cancel a class that had actually committed. I audited every booking, cancel and reschedule path, added idempotency keys to all of them, and made each path verify whether the insert committed before compensating. Hours can no longer be double-spent or wrongly refunded, even on network failures and retries.

**18 Aug 2026 - Go-live cutover**
The business moved off its third-party classroom platform onto this system, with real students, teachers and schedules from day one. The cutover was gated on a full test plan pass across all three portals. The platform has run in production since, and every change after this date is live-system work.

**11 Aug 2026 - Cron reporting healthy while failing**
A scheduled health-check job returned success even when its database query failed, so the schedule log showed green while the job did nothing. I bound the error so a failing run returns a failure status, and later gave every scheduled job a heartbeat write so a single query proves each one fired. Silent scheduled failures are now impossible to miss.

**04 Aug 2026 - Dead client-side error monitoring**
Browser errors had not been reaching the monitoring service, because the client DSN lacked the framework's public environment prefix, and nothing looked wrong. I moved the client init to the correct instrumentation file, fixed the variable, and verified events actually arrived. The standing rule since: monitoring needs proof of receipt, not just configuration.

**29 Jul 2026 - Timezone rendering audit**
Multiple pages were rendering dates and times in the browser's timezone instead of the account's timezone, shifting labels by a day for users west of UTC. I fixed every affected component in one pass, threading the account timezone through each formatter and verifying with multi-timezone parity harnesses. Constructing local dates via ISO string conversion was banned project-wide.

**29 Jul 2026 - Function recreate resets execute grants**
Recreating a database function via drop-and-create silently restored execute permission to roles that had been deliberately revoked. I caught it in an audit and added an explicit re-revoke and re-grant step to the standing migration checklist for every function recreate. Least-privilege on database functions now survives routine maintenance.

**07 Jul 2026 - Closed a student-forgeable no-show path**
The write policies on the lessons table would have allowed a student session to record a teacher no-show, which affects teacher pay. I moved the lesson insert to a server-side privileged client and dropped the student write policies and residual grants entirely. Pay-affecting records can now only be created by trusted server code.

**19 Jun 2026 - Column grants fail silently**
A select-all query against a table carrying any column-level revoke returned null rows with no error at all, which cost hours to diagnose the first time. I switched every sensitive table to explicit column lists and added grant verification after every schema change. A permissions problem now fails loudly instead of masquerading as missing data.

**16 Jun 2026 - Cron paying for unreported classes**
A scheduled job was automatically marking lessons complete, which made them billable, even when the teacher had never filed the required class report. I disabled it, and later replaced it with a report-driven lifecycle where an unreported class becomes missed and forfeits pay instead. Teacher pay is now always backed by a filed report.

**09 May 2026 - Orphaned video meetings**
Cancelling or rescheduling a lesson deleted the booking but left its video meeting alive in the calendar system. I added meeting cancellation to every cancel and reschedule path across all three portals, made the cleanup idempotent so a repeat delete cannot error, and wrote a retroactive cleanup script for existing orphans. Cancelled classes now leave nothing behind.

**03 May 2026 - Fail-safe defaults**
A banner prompting users to complete their profile was hidden whenever its flag was null or undefined, so exactly the users who needed it never saw it. I flipped the fallback so an unknown state shows the prompt rather than hiding it. The standing rule since: when a value is missing, default to the state that prompts the user to act, never the one that hides the problem.
