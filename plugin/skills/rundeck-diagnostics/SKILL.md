---
name: rundeck-diagnostics
description: >-
  Run diagnostic runbooks in Rundeck to gather point-in-time system state
  during an investigation — current process and service health, disk or
  resource pressure, live log tails, connectivity checks — the evidence
  telemetry alone cannot provide. Use whenever a hypothesis needs verifying
  against the actual system, when an alert names a host or service with
  Rundeck diagnostics available, or at the start of a new incident to rule
  out an active SaaS vendor outage with vendor-status-check. Not for
  classifying or re-reading evidence already gathered, and the incident-start
  check is a once-per-incident opener rather than a step to repeat for every
  later subtask. Approved runbooks live in the diagnostics-demo project (the
  default); users can point at other projects explicitly.
---

# Rundeck diagnostics

Rundeck holds this organisation's codified diagnostic runbooks. They are
read-only checks — they inspect system state, never change it — so running
them is always safe. Access is via the **Rundeck MCP connector**.

**Approved runbooks live in the `diagnostics-demo` project** (group `diagnostics`,
tagged `diagnostic`). Default to it for every lookup and run; only use another
project when the user names one explicitly.

## What this needs

Two groups of tools, and which are bound decides what this skill can do:

| Need | Tools |
|---|---|
| Find and inspect jobs | `list_projects`, `list_jobs`, `get_job`, `get_job_definition` |
| Read what past runs found | `list_executions`, `get_execution`, `get_execution_state`, `get_execution_output` |
| Execute a diagnostic | `run_diagnostic` |

**Run diagnostics with `run_diagnostic`, never `run_job` or
`run_job_and_wait`.** `run_diagnostic` only runs jobs in the
`diagnostics-demo` project's `diagnostics` group, all of which are read-only
checks, so it is a read tool and is bound in investigations as well as when a
person is talking to the agent. `run_job` and `run_job_and_wait` write: they
are never bound in an investigation, and may be switched off entirely.

**Read the session's available tools before planning a run.** Where
`run_diagnostic` is absent, this skill is in **read-only mode**:

- Do not call or retry `run_diagnostic`, `run_job` or `run_job_and_wait`, do
  not describe a hypothetical run, and never assume what a run would have
  returned.
- `list_executions(job_id)` and `get_execution_output(execution_id)` still
  work. A recent run of the right job is real evidence — cite it with its
  timestamp, and say plainly where it predates the incident.
- Where no recent run covers the question, report the check as **unrun**, not
  as a negative finding. "The disk-usage diagnostic was not run" is honest;
  "the disk is not full" is not.
- Binding `run_diagnostic` is an administrator's job in the dashboard.
  Record the gap and carry on — never offer it as a remediation step to
  responders mid-incident.

## First move on a new incident

**Where `run_diagnostic` is bound**, run
`run_diagnostic(job="vendor-status-check", options={"vendor": "all"})` before
forming hypotheses — it checks the public status of major SaaS vendors and
the Paylinity payment provider in ~5 seconds. An active vendor outage
reframes the whole investigation; ruling one out is the cheapest evidence you
can gather. Cite its summary either way ("no active vendor issues" is a
finding). A vendor reported degraded is a lead to test against the incident's
symptoms, not a root cause by itself.

The `vendor=all` run already covers every vendor, including any the alert
names, so don't follow it with a single-vendor run during the opener. Re-run
with one vendor only later, when a new question needs a fresh reading the
first run can't answer — for example, checking whether a vendor degraded in
the first run has since recovered. The `vendor` option takes exact lowercase
values: `all`, `paylinity`, `slack`, `zoom`, `github`, `jira`,
`google-workspace`, `cloudflare`, `datadog`, `stripe`, `twilio`, `openai`.

**Where it is not**, check `list_executions` for a recent `vendor-status-check`
run and cite that if one covers the incident window. Otherwise record vendor
status as unchecked and move on.

Either way this is a once-per-incident opener: later subtasks in the same
investigation do not repeat it.

## When to reach for this

- A hypothesis needs verification against live system state ("is the disk
  actually full?", "is the service process running?").
- Telemetry sources don't cover the affected component, or data is delayed.
- The alert or incident names a specific host or service and you want its
  current health, not its history.

Prefer native telemetry sources for metrics/log *history*; use Rundeck for
*point-in-time system state* and checks that only run inside the environment.

## How to run a diagnostic

1. **`list_jobs(query={project: "diagnostics-demo"})`** — see what's available
   (only look elsewhere if the user named a different project; `list_projects()`
   shows what exists). Job descriptions state when each check is useful and
   what it returns. Don't guess job ids.
2. **Map incident context to options.** Check the option schema with
   **`get_job(job_id)`** — options use plain names (`service`, `target`,
   `window_minutes`), many with enforced allowed values. Take values from the
   alert payload, affected catalog entries, or earlier findings. If a required
   option can't be inferred, say so rather than inventing a value.
3. **`run_diagnostic(job="<name>", options={...})`** — runs the job, waits
   up to ~25 seconds for it to finish, and returns the final status, output
   (including the summary block), and a permalink in one call. It refuses
   any job outside `diagnostics-demo`/`diagnostics`; that refusal is final,
   so don't retry it or fall back to `run_job`. If `run_diagnostic` isn't
   bound, you are in read-only mode (above): read past executions rather
   than attempting a run.
4. If the result says the job is still **running**, you must call
   **`get_execution_output(execution_id)`** before drawing any conclusion.
   Never cite an in-flight or timed-out run as evidence.

## Reading the output

- Each job ends with a `=== DIAGNOSTIC SUMMARY ===` block — that is the
  authoritative conclusion; the lines above it are supporting detail.
- A **failed** execution usually means the check itself could not run
  (missing host, bad option), not that the system is unhealthy. Treat it as
  "no evidence", report why, and consider a different diagnostic — don't
  fold a failed run into your hypothesis either way.
- Always include the execution permalink when citing a diagnostic in
  findings, so responders can see the full output.

## Hard rules

- Diagnostics gather evidence. They are **never** a fix, and their success
  does not mean the incident is resolved.
- Do not retry a diagnostic more than twice; repeated failure is a finding
  in itself ("could not verify X because the disk-usage check errors").
- If no available diagnostic fits, say so explicitly rather than running a
  loosely related one and over-interpreting its output.
