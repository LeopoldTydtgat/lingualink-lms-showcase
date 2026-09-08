# ADR 3: Monitoring and error tracking

**Status:** Adopted, in production
**Date:** 2026

## Context

The platform is operated by one person and used daily by real students and teachers. There is no ops team watching dashboards. If something breaks silently - a failed booking, a dead cron job, an email that never sends - the first report would otherwise come from a paying user, which is the worst possible monitoring system.

The goal: errors surface to the operator before users need to report them, on both the server and in the browser.

## Decision

Sentry as the single error-tracking system, wired into every layer of the application.

- Server-side errors are captured across API routes and server rendering, including a framework-level hook so errors in request handling are reported even when no local try/catch exists.
- Client-side (browser) errors are captured separately, because a working server tells you nothing about a broken browser experience.
- Scheduled jobs (reminders, overdue-report flagging, calendar sync) are treated as first-class monitored surfaces: a cron that stops running is an incident, not a mystery.
- Errors carry context (which route, which portal) but not sensitive personal data.

## Alternatives considered

- **Log files and hosting-platform logs only.** Logs are pull, not push: they answer questions you already know to ask. Rejected as the primary system because nobody reads logs unprompted.
- **Uptime pinging only.** Detects "site down", misses "site up but bookings failing". Rejected as insufficient alone.
- **Building alerting in-house.** Email-on-error from within the app. Rejected: the app reporting on its own failure is fragile exactly when it matters, and the build time was not justified.

## Consequences

**Good**
- Failures announce themselves. Several production issues were found and fixed from Sentry alerts before any user reported them.
- Browser-side coverage catches the class of bug a server monitor can never see.

**Costs and sharp edges (learned in production)**
- **Monitoring can die silently, which is the one failure it cannot report.** The browser-side setup shipped for weeks with a dead configuration: the monitoring key was missing the framework's public env prefix, so the client bundle built and ran with monitoring silently disabled. Nothing errored, because sending nothing is not an error.
- The lesson that stuck: **configuration is not proof.** The standing rule is now proof of receipt - after any change to the monitoring setup, trigger a deliberate test error on each surface and confirm it arrives, in production, before considering the change done.
- Error tracking needs curation. Noisy or expected errors must be filtered, or real alerts drown.

## What I would tell someone building this

Ask one question of your monitoring: "if this exact component died right now, what would tell me?" Walk every surface - server, browser, each scheduled job - and make sure the answer is never "a user". Then break each one on purpose and watch the alert arrive. A monitoring system you have never seen fire is a hope, not a system.
