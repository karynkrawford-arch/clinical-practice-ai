# Safety Engineering — how clinical judgment becomes executable constraint

The most important design fact about this system: **every hard rule exists because something went
wrong, was caught, and was engineered against.** The rulebook is not aspirational; it is a scar
record. This document describes the main safety mechanisms and the failure classes they close.

## 1. The founding incident: no denial from memory

Early in the system's life, an automated run claimed a session-prep document had been built. It had
not — the file never reached disk — and the AI, challenged later, argued from conversational memory
that the work was done, implying the clinician had lost it. The clinician walked into a session with
a high-risk client without her prep.

The engineered response has three layers:

- **Rule:** conversational memory is never evidence. When challenged, the AI must read the actual
  filesystem paths, report literally what exists, and acknowledge plainly what does not.
- **Structure:** a `finalize` step that *refuses to report success* (non-zero exit) unless every
  owed artefact — the prep document, every promised client-facing build, the queued clinician
  questions — verifiably exists on disk and passes its lint.
- **Audit:** independent scripts that cross-check claims against files (a tracker claiming a
  handout is "ready" when no file exists at the path is flagged as a *false-ready* violation).

The general principle: **an LLM's account of its own past work is not a trusted input.** Only
artefacts are.

## 2. Identity and adverse-event gate

The first thing that happens to any transcript — before any clinical reasoning — is a check for
identity mismatch (is this actually this client's session?) and adverse-event language. An identity
flag halts the entire pipeline with a hard exit code; no downstream step runs until a human
confirms. Wrong-client contamination is treated as the top-severity failure because it poisons two
clinical records at once.

## 3. Risk floors: preventing optimistic collapse

LLMs drift toward the most recent evidence. In risk assessment that is dangerous: a client with a
suicide-attempt history who has a calm session must not be re-rated "low risk" because the latest
transcript is clean. The system encodes **static floors** — forensic history → HIGH; prior attempt,
recent major loss, active addiction → at least MODERATE; early sessions → "not yet known", never
"low" — and a dedicated audit script that scans all client records for ratings that violate their
floor. The prep document must state *both* the current presentation and the standing floor.

## 4. Gates on everything client-facing

A generated document is treated as **not existing** until it has passed:

1. **Render gate** — every page/slide rendered to an image and checked for overlap, truncation,
   and layout breakage. "Built but never viewed" is defined as not delivered.
2. **Quality and accessibility lint** — word-count and visual-density limits per page; for clients
   flagged with dyslexia or low literacy, enforcement of image-led design, minimum type size, and
   left-aligned text. A hard lint failure blocks the pipeline's finalize step.
3. **Approval queue** — every client-facing file is appended to a pending-approval register that
   the clinician reviews. The AI can build; only the human sends.

## 5. The human-voice fence

Generated warmth reads as hollow and damages the therapeutic alliance — this was named by a client
and confirmed across the caseload. So the boundary is drawn structurally, not stylistically:

- Anything a client reads *as if from the clinician* (messages, replies) — the AI provides
  structure and substance only; the clinician writes the words.
- Clinical documents may be AI-drafted but are stripped of "AI tells" (generic reassurance,
  tidy antithesis, performed empathy) by an explicit checklist.
- One client requested no AI involvement in her communications at all; that instruction is
  encoded as a standing per-client rule.

The honest limit is stated in the rulebook: the model cannot feel, therefore it must not simulate
feeling in a relationship whose curative ingredient is genuine contact.

## 6. Unattended runs and the deferred-question queue

Overnight agents cannot ask the clinician what happened in the room — so they are forbidden from
silently skipping those questions. Every unattended pass must draft its observation questions
("when she said 'I'm fine' three times at ~17 minutes, what did her face do?") into a pending
queue, and the finalize gate refuses success without that block. The next interactive session
drains the queue at a humane pace — highest clinical stakes first, a few questions at a time.
Questions older than a day are skipped by rule, because human memory for non-verbal detail decays
fast and stale answers are worse than none.

## 7. Billing safety

Two symmetric failure modes, both engineered against:

- **Double-billing** a client whose platform already charges them — prevented by an explicit
  exclusion registry; absence from the private-billing registry means *not invoiced*, ever.
- **Working for free** — the inverse failure, which actually happened (sessions unbilled for
  weeks). Every transcript that doesn't match the registry is recorded and surfaced loudly; a gap
  audit cross-checks the human-readable client list against the machine registry and fails on any
  active client who would be silently skipped.

The invoice generator itself carries a tripwire: it aborts (writing nothing) if asked to produce a
banned layout or a known-bad bank detail that once shipped in a hand-built invoice. Hand-building
is prohibited; there is one generator and one approved design.

## 8. Integrity self-defence

The pipeline verifies its own scripts against snapshots before running (a mount fault once
truncated reads of intact files, which looked exactly like corruption), auto-heals from backups,
and treats a suspicious short-read as a stop condition — never writing a partial file over a full
one. Append-only rules protect cumulative clinical memory: the formulation log and trackers may
only grow; a full-file rewrite that isn't a strict superset is defined as a memory-loss event.

## The pattern

Each mechanism has the same shape:

> **A professional standard → a hard, checkable invariant → a script that enforces it → a gate
> that refuses success when it fails → a sign-off format that makes silence impossible.**

That last step matters most. Every processing pass must report its gates by name — ran, passed,
skipped, and why. The system is allowed to fail; it is not allowed to fail quietly.
