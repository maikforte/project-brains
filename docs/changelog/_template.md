# YYYY-MM-DD — Title that says what changed

**Features:** {{primary-feature}}{{, other-feature-touched}}

{{One or two sentences: the request or problem, and where it came from (ticket, user report, decision).}}

## What happened / Cause

{{For a fix: the symptom, then the root cause. Include the evidence that proved it, and any first diagnosis that turned out wrong.}}
{{For a feature: what was asked for and the constraints that shaped it.}}

## Changes

- {{What now works, in concrete terms}}
- {{Files: `src/features/.../x.ts`, migration `supabase/migrations/YYYYMMDDHHMMSS_name.sql`}}
- {{Permission codes added: `feature.action`}}
- {{Components reused or extended (CLAUDE.md rule 10), or what was checked before creating new ones}}
- {{Manual: `docs/manual/<feature>.md` updated, or why not}}

## Tests

- {{`path/to/name.test.ts`: what it covers}}
- {{For a fix: confirmed the regression test failed before the fix}}
- {{Or: none, because ...}}

## Rejected

- **{{Option}}.** {{Why it lost.}}

## Still open

- {{Known gap, follow-up, or assumption not yet verified}}
