# Architecture — a document-native clinical operations system

No database, no web stack, no cloud storage of clinical content. The system is deliberately built
on **files, scripts, and gates**, because files are what a clinician can open, verify, and own —
and because the "no client data leaves the machine" constraint rules out most conventional
architectures from the start.

## The data layer: living documents per client

Each client is a folder with a small set of canonical living documents:

| Document | Role | Update discipline |
|---|---|---|
| Case file | The *person*: living life-circumstances portrait, diagnosis, attachment, risk profile — plus a cumulative **clinical formulation log** | Formulation log is append-only, dated entries; never overwritten |
| Session prep ("SNAP") | The *session cycle*: what happened, what it means, exactly how to open and run the next session, safety box on top | Exactly one live copy; predecessors auto-superseded |
| Psychoeducation tracker | What teaching material has been delivered, queued, or ruled out, and why | Claims audited against files on disk |
| Discussion log | Per-client working memory for the AI: a pinned living case conceptualisation plus dated entries | Auto-appended whenever the client is discussed |

Every clinical markdown file has an auto-generated Word mirror — the clinician reads Word; the
pipeline reads text. A scheduled sync keeps the two honest, and hand-editing the mirror is
prohibited (it would be silently overwritten).

Two structural privacy rules shape all naming: client-facing folders never carry surnames or ages
(a neutral code scheme with a local, non-syncing key file maps codes to identities), and nothing
client-identifying may exist inside any cloud-synced path.

## The pipeline: one script owns the plumbing

All transcript processing flows through a single orchestrator with two mandatory bookends:

```
prepare  →  [clinical work: extraction, updates, builds]  →  finalize
```

- **`prepare`** locates every canonical file, runs the identity/adverse-event gate, detects
  promises made in-session (both the clinician's "I'll send you X" and implied teaching
  materials), and prints the checklist of what this pass owes.
- **The clinical middle** is the AI's reasoning work — but its outputs are all files at canonical
  paths, so they are checkable.
- **`finalize`** is the enforcement point: it verifies every owed artefact exists and passes its
  lint, then — and only then — performs the destructive moves (supersede the old prep, archive the
  transcript). A non-zero exit means the session is not processed, and nothing destructive has
  happened.

The separation of concerns is strict and deliberate: **the model owns clinical thinking; scripts
own file plumbing.** The model never hand-moves clinical files. This one decision eliminated the
system's most repeated early failure — documents that "existed" in conversation but never reached
disk.

## The build layer: generators, not generations

Client-facing materials are produced by ~150 purpose-built generator scripts (Node.js with docx/
pptx libraries, Python with reportlab/python-pptx, PowerShell for OS glue) rather than by free
generation each time. Key properties:

- **Templates are gold standards.** A personalised deck is built by cloning a benchmark file and
  filling it — never regenerated from scratch, which drifts in design.
- **Content-aware layout.** Generators measure wrapped text and advance by actual rendered height;
  fixed-box layouts are banned because they truncate.
- **A reusable visual language.** An original "box figure" character library (generated
  programmatically, one engine, exported to a shared PNG library) carries emotion across all
  teaching materials, so decks teach through image rather than text — which also serves the
  accessibility rules for dyslexic and low-literacy clients.

## The scheduling layer: unattended but constrained

Scheduled agents handle the recurring load: overnight and evening transcript processing, a nightly
next-day digest (pending approvals on top, then safety-relevant rows, then prep), retention watch
(cadence gaps against true session dates), billing audit, feedback-cadence audit, and file-hygiene
sweeps. Design constraints for anything unattended:

- Draft-only: unattended runs produce first-pass work for human review; nothing client-facing is
  ever auto-sent.
- Queue, never skip: questions requiring human observation are queued with a gate that makes
  skipping impossible to hide.
- Loud degradation: a run that cannot complete a step must name it at the top of its output — a
  silent deferral is a rule violation, not a judgment call.

## The interaction layer

- A **native Windows client panel** (PySide6/Qt) shows the active caseload with a work-status
  traffic light per client (red = something owed: unprocessed session, file awaiting approval,
  open promise) and one-click entry into a per-client AI workspace.
- **Per-client workspaces** enforce a "history first" rule: the AI must reload the client's
  discussion log, conceptualisation, case file, and latest prep before saying anything clinical —
  no cold takes on a person it has known for thirty sessions.
- Desktop status files and launchers give the clinician glanceable state (action queue, retention
  watch, uninvoiced alerts) without opening any tool.

## The rulebook as kernel

The whole system is governed by a long-form constitution (CLAUDE.md) that the AI loads every
session. It contains the clinical protocol (a staged therapy pathway with retention as the prime
directive), the hard rules (each one traceable to a documented failure), the filing system, the
gate definitions, and the required sign-off formats. Its evolution is itself governed: rules are
added with their rationale, compacted periodically without loss (verified by audit), and one
section lost to disk corruption is explicitly marked as unrecoverable rather than silently
reinvented.

That last detail is the architecture in miniature: **the system prefers a visible gap over a
plausible fabrication.**
