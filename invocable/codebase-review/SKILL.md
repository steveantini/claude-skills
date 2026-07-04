---
name: codebase-review
description: A structured four-pass read-only audit for finding real bugs, security holes, architecture drift, and test/doc gaps in a web app (frontend + backend + database). Use whenever the user asks for a codebase review, code audit, bug hunt, architecture review, security review, dependency/config review, test-quality review, or a pre-launch / pre-scaling / pre-investor health check, and also when they ask you to "look the project over," "find what's broken before we ship," or "make sure this is safe to open up to more users," even if they don't say the word "audit."
---

# Codebase review: the four-pass audit

A disciplined way to review a whole codebase for a web app (frontend, backend,
database). It is read-only first, ranked, and swept by CLASS. It was distilled
from a real audit cycle that found production bugs which passing tests and
confident docs had hidden.

The shape: four read-only passes (backend, frontend, architecture, tests+docs),
each producing a ranked findings report. Then fix worst-first, one change-set at
a time, and encode each fixed invariant so the whole class stays dead.

## Principles (read before starting any pass)

- **Audit read-only; never fix mid-pass.** Finding and fixing at once makes you
  stop hunting the moment you find something. Report the whole pass, then fix.
- **Rank every finding** P0 / P1 / P2 / P3 and put P0s at the very top:
  - **P0** exploitable or data-loss RIGHT NOW (auth bypass, cross-user leak,
    secret exposure, silent corruption). Flag loudly.
  - **P1** a real bug that bites soon under normal use.
  - **P2** a latent trap that bites under a specific but plausible condition.
  - **P3** hygiene / maintainability, no runtime impact yet.
- **Every finding states:** title, `file:line`, 2-3 sentences on the risk (the
  concrete failure, not just "this is bad"), and a one-line fix direction.
- **Sweep for the CLASS, not the instance.** One bug is a lead, not a
  conclusion. Before closing it, grep the whole repo for its siblings, they are
  almost always there. A single audit's most valuable output is usually "there
  were nine of these, not one."
- **When you fix, kill the class.** Encode the invariant into the project's
  CLAUDE.md (or equivalent house-rules doc) so a future session cannot
  reintroduce it. A fix that lives only in one file is half a fix.
- **State judgment calls explicitly.** In every report, say what you
  deliberately did NOT change and why (e.g. "the fakes teach the wrong contract
  but mask no live bug because the real repos are guarded, so aligned-not-
  rewritten"). Silent non-decisions are how the next reviewer redoes your work.
- **Verify structural claims independently; trust neither the docs nor the
  prompt.** Audits repeatedly find both wrong: a doc says a migration ran, the
  code says otherwise; the prompt asserts "X is handled," the code disagrees.
  Open the file and confirm before you rank.
- **Fix a pass's P1s before running the next pass.** A live P1 changes what
  later passes should worry about, and stacking unfixed criticals across four
  passes loses them.

## The four passes

Run them in order. Each is a read-only sweep ending in a ranked report. The
checklists below are the items that actually caught bugs, not an exhaustive
theory of everything.

### Pass 1 — Backend correctness and security

- **External-call return shapes.** Every place code assumes the shape of a
  DB/HTTP/SDK response. The seminal bug: a client method that returns `None`
  (not an object-with-null) on the empty case, then code does `.data` /
  `[0]` / attribute access on it. Grep for the access pattern, not the one
  crash you saw.
- **Auth coverage per route.** Enumerate every route and confirm each has the
  auth gate it needs. Look for the one handler that forgot the dependency.
- **Cross-user scoping on privileged reads.** Anywhere a service-role /
  admin / RLS-bypassing client reads data: confirm it is scoped to the right
  user or gated to operators. A metrics or observability endpoint that reads
  across users without a gate is a P0/P1 leak.
- **Secrets and content in logs and errors.** Grep for logging of keys, tokens,
  raw prompts/responses, PII; and for provider/DB error text forwarded verbatim
  to the client (leaks internals and aids attackers).
- **Input validation** on every user-facing endpoint (body, query, path).
- **Rate limits on expensive endpoints**, especially anything that spends money
  (AI / paid-API calls) or is unauthenticated.
- **Timeouts on every external call.** An un-timed upstream call can wedge a
  worker indefinitely; confirm a bound exists and is sane.
- **Unbounded in-process state.** Caches / rate-limiter maps keyed by
  attacker-controlled input (IP, email, arbitrary id) with no size cap are a
  memory-exhaustion vector. Confirm eviction / max-keys.
- **Background-task failure modes.** What happens to an in-flight job on
  crash/redeploy/SIGKILL? Does it strand a row in a non-terminal state? Is there
  a reconciler / finally-net?

### Pass 2 — Frontend correctness

- **Effect cleanup and cancellation.** Every subscription / timer / fetch in an
  effect needs teardown; a late callback after unmount is a bug.
- **Stale-response races on navigation.** When the route param changes mid-
  flight, the older in-flight response can overwrite the newer view. Prefer
  keying the component/instance by id (so React remounts) over hand-rolled
  supersession guards; note which the code uses.
- **Unhandled async throws** that escape error boundaries (boundaries catch
  render errors, not promise rejections). Confirm async paths have `.catch`.
- **Silent failure states.** The trap: an error path renders the same UI as the
  empty/success path (a failed load shows "no results" instead of an error), so
  users and you never see the failure. Errors must look like errors.
- **Unbounded loops / pagination.** A "load all pages" loop with no cap, or a
  client filter over an unbounded set. Confirm a ceiling and a note when it is
  hit.
- **Auth-flash states.** Does a protected view flash content before the auth
  check resolves? Does a signed-out user momentarily see signed-in chrome?
- **Client storage and console leakage.** Tokens/PII in localStorage or
  console.log left in production.
- **Third-party scripts in layouts.** Any capture/analytics/experiment script
  loaded app-wide, especially one left over from a spike.

### Pass 3 — Architecture and configuration

- **Dependency pinning and reproducible deploys.** Are deps exact-pinned with a
  lockfile, or can a rebuild silently pull a new major? Unpinned deps make a
  deploy non-reproducible and break with zero code change.
- **Fail-open defaults.** Any env var whose UNSET value enables the less-safe
  behavior. The rule: unset must behave as production (docs hidden, detail
  suppressed, feature locked). Only an explicit `development` opens things up.
- **Enum / type parity across the stack.** Status enums, DTOs, and shared
  constants that exist in both frontend and backend and can drift. A status the
  backend can emit but the frontend doesn't handle is a latent bug.
- **Error-status correctness.** Not-found returns 404 (not 500 or a generic AI/
  internal error), validation returns 4xx, auth returns 401/403. Wrong codes
  mislead clients and monitoring.
- **Duplicated auth gates that can drift.** Two functions/decorators that both
  gate the same privilege will eventually disagree. Standardize on one; retire
  the other.
- **Dead code.** Unimported modules, retired components, one-off scripts.
  Deleting them shrinks the audit surface and removes traps.
- **Docs-vs-code drift.** Spot-check load-bearing doc claims against the code
  (a migration "applied," a flag "enabled," a route "removed"). Drift here
  misdirects the next session.
- **Single-replica / in-process state inventory.** BEFORE anyone scales
  horizontally, list every piece of in-process state (rate limiters, caches,
  registries, capability maps) and note whether multi-replica breaks it LOUDLY
  or SILENTLY. Silent breakage (N-times rate budget, stale revocation) is the
  dangerous kind.

### Pass 4 — Tests and documentation

- **Fakes that model the wrong contract (masking).** The most dangerous test
  smell: a mock/fake whose return shape differs from the real dependency, so
  tests pass while the real code would crash. Check every hand-rolled fake of an
  external client against the library's ACTUAL contract (the empty case
  especially).
- **Critical paths with zero direct tests, ranked by blast radius, auth first.**
  List the behaviors that would be catastrophic if wrong and have no direct
  test. Auth/authz gates, encryption round-trips, and terminal-state writes rank
  above feature logic.
- **Test hygiene.** Shared in-process state not reset between tests (order
  dependence); real secrets/`.env` loading into the test process; tests that
  pass only because of leaked state from a prior test. Confirm teardown clears
  singletons and env is stubbed.
- **Doc-vs-doc contradictions.** Two docs that disagree about production state
  (one says a backfill ran, another says pending) can misdirect a fresh session
  into touching real data. Reconcile to the verified truth and keep an honest
  caveat where something is asserted-but-unverified.
- **Can the deploy runbook alone stand up a working environment?** Cross-check
  the runbook's env table against every config field the app actually reads. A
  required, fail-closed secret missing from the runbook (e.g. an encryption KEK
  the write path needs) means a new environment built from the doc is broken on
  arrival.

## Output conventions

- **Each pass** produces a ranked findings report: P0s first and flagged, then
  P1/P2/P3, every finding with title, `file:line`, 2-3 sentence risk, one-line
  fix direction. End the report with a **3-sentence health assessment** (overall
  posture, the single most important thing to fix, and your confidence level).
- **Pass 4 additionally** produces two lists: a **top-5 untested critical
  behaviors** list (ranked by blast radius), and a **continuity-gap list** (the
  doc contradictions / runbook holes a fresh session could trip over).
- **When fixing** (after the audit): worst-first, ONE change-set at a time, run
  the test suite between change-sets, and for each fixed class add or update the
  invariant note in the house-rules doc so it stays dead.

## Cadence

- **Full four-pass review** after each major arc ships, or quarterly, whichever
  comes first.
- **Always before expanding access** (opening a beta wider, going public,
  onboarding the first outside users). New users are new attack surface and new
  blast radius; run all four passes first, weighting pass 1 (security) and the
  cross-user-scoping and rate-limit items.
- **Lightweight weekly** production-log review between full audits: scan startup
  lines (did every subsystem wire up?), error tracebacks (new classes?), and
  request patterns (abuse, unexpected volume, slow endpoints). This is the cheap
  early-warning layer that tells you when to pull a full audit forward.

## Improving this skill

At the end of a full review, once the final pass's fixes are banked, reflect on
whether the review PROCESS itself (not the project's findings) revealed a way to
improve this skill: a checklist item that caught nothing across the whole review
and just wastes effort, a bug class you hit that no pass hunts for, a better
pass/item ordering, or a sharper output convention.

If something surfaced: propose the specific edit to the founder. Quote the exact
wording to add / change / remove and say why. Then ask for confirmation and wait
for it. NEVER edit this file without explicit confirmation, the whole point is
that the skill only changes deliberately.

Strict criteria for a proposal:
- **Process-level and project-agnostic only.** A finding tied to one project's
  stack, schema, or history belongs in THAT project's CLAUDE.md (or house-rules
  doc), not here. If you are unsure which it is, treat it as project-specific and
  leave this skill alone.
- **Prefer removing or tightening over adding.** This skill earns its keep by
  staying tight and scannable; a new bullet has to displace its own weight. A
  checklist item that reliably catches nothing is a better edit to propose than
  any addition.

If no process-level improvement surfaced, say so in one line and move on.
