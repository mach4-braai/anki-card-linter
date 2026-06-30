# Anki Card Linter — Product Requirements Document

**Date:** 2026-06-30
**Status:** Draft
**One-liner:** A critique-only Anki add-on: a bring-your-own-key LLM lints
hand-written cards against the Minimum Information Principle and Matuschak's
prompt properties, flagging violations without ever authoring the card.

## Overview

Spaced repetition only works as well as the prompts you feed it. Most people
write bad prompts and never find out why — cards that bundle five facts, ask
vague questions, or can be answered without retrieval quietly waste review time
for years. The two best-known rubrics for diagnosing this (Wozniak's *Twenty
Rules of Formulating Knowledge* and Matuschak's *How to write good prompts*)
exist as essays, not tooling. Nothing checks your actual deck against them.

Anki Card Linter is a **critic, not an author**. It reads a card, runs it
through an LLM against a fixed rubric, and tells you which principles the card
violates and why — in the card's own terms. It never rewrites the card, never
fills a field, never auto-creates anything. The human keeps authorship; the tool
supplies judgment.

This boundary is the product. A tool that writes cards for you defeats the
purpose of spaced repetition (the encoding effort is where the learning starts)
and produces generic, un-owned prompts. A tool that only critiques makes you a
better card author every time you use it.

## Goals

- Diagnose a single card, or a batch of selected cards, against a fixed,
  well-known rubric.
- Return **specific, actionable, per-principle** findings — which rule, where,
  why, and a concrete suggestion the human can choose to apply themselves.
- Run entirely on the user's own LLM API key. No hosted backend, no telemetry,
  no card content leaving the user's machine except to the LLM endpoint they
  configured.
- Be unmistakably read-only: the add-on holds no write path to the collection.

## Non-goals

- **Authoring or auto-fixing cards.** The add-on never writes to a note. Not as
  an opt-in, not "just the obvious ones." Suggestions are text the user reads.
- **Scheduling, FSRS tuning, or review-time behavior.** Out of scope; this is an
  authoring-quality tool, not a scheduler.
- **Bundling an API key or proxying through a shared service.** Strictly
  bring-your-own-key.
- **Grammar/spell-checking.** The rubric is about prompt *structure and
  retrieval design*, not prose correctness.
- **A custom rubric editor (v1).** The rubric is fixed and curated. User-defined
  rules are a possible v2 (see Future work).

## Users & motivation

- **The serious self-learner** with a large hand-written deck who suspects some
  cards "don't stick" but can't tell which or why.
- **The new SRS user** who has read (or half-read) the rubrics and wants a
  feedback loop while the habits are forming.
- **The deck author** publishing shared decks who wants a quality pass before
  release.

All three share one need: *targeted* feedback on *their* cards, not another
essay telling them to be focused.

## The rubric (what "good" means)

The linter checks against two canonical, citable sources. These are the
acceptance rubric — findings MUST map to a named principle below.

### Matuschak — properties of effective retrieval-practice prompts

A prompt should be:

| Property | Violation the linter flags |
|----------|----------------------------|
| **Focused** | Question or answer carries multiple details; retrieval target is diffuse. |
| **Precise** | Vague question that admits many valid answers ("Tell me about X"). |
| **Consistent** | Answer would differ run-to-run; invites retrieval-induced forgetting. |
| **Tractable** | So hard it won't be reliably answered; needs decomposition or a cue. |
| **Effortful** | Answer is trivially inferable from the question; no real retrieval. |

Source: Andy Matuschak, *How to write good prompts* (2020),
https://andymatuschak.org/prompts/

### Wozniak — Twenty Rules of Formulating Knowledge (linting subset)

The high-signal, machine-detectable rules:

| Rule | Violation the linter flags |
|------|----------------------------|
| **#4 Minimum information principle** | Card crams compound facts that should be several atomic cards. |
| **#1 Don't learn what you don't understand** | Card memorizes opaque strings with no comprehension hook. |
| **#9 Avoid sets** | Answer is an unordered set ("name all the…"). |
| **#10 Avoid enumerations** | Answer is a long ordered list; suggest cloze/overlap. |
| **#11 Combat interference** | Card is near-duplicate / easily confused with a sibling concept. |
| **#12 Optimize wording** | Wordy stem; the retrieval cue is buried in filler. |

Source: Piotr Wozniak, *Effective learning: Twenty rules of formulating
knowledge* (1999),
https://www.supermemo.com/en/blog/twenty-rules-of-formulating-knowledge

The Minimum Information Principle is the headline check — it is both the most
violated and the highest-leverage rule, and it subsumes "focused" + "tractable"
from Matuschak's list.

## User experience

### Two entry points

```
┌─ Editor ───────────────────────┐     ┌─ Browser ──────────────────────┐
│  [Front] ...                    │     │  ▣ card 1                       │
│  [Back]  ...               [✦]  │     │  ▣ card 2   → right-click       │
│                             │   │     │  ▣ card 3     "Lint selected"   │
└─────────────────────────────┼──┘     └────────────────┼───────────────┘
                              click                     N notes
                                │                         │
                                ▼                         ▼
                     ┌──────────────────────────────────────┐
                     │  LLM call (user's key, background)     │
                     └──────────────────┬───────────────────┘
                                        ▼
                     ┌──────────────────────────────────────┐
                     │  Findings dialog (read-only)           │
                     │   • Front/Back echoed                  │
                     │   • per-principle findings + why       │
                     │   • suggestion text (copyable)         │
                     └──────────────────────────────────────┘
```

1. **Single-card, in-editor.** A button (`✦`) in the editor toolbar lints the
   card currently being written. Closes the author's feedback loop at the moment
   of authoring.
2. **Bulk, in-browser.** A "Lint selected cards" context-menu action runs the
   selected notes and presents a per-card report.

### Output

A scrollable, read-only findings panel. For each card:

- The card's fields, echoed for context.
- A **verdict** (e.g. `3 findings`) — or `looks good` when nothing fires.
- Per finding: the **principle name** (e.g. "Minimum information principle"),
  the **specific problem** in this card, and a **concrete suggestion** the user
  may choose to act on manually.
- Suggestions are copyable text. There is no "apply" button — by design.

### Configuration

A config screen captures the user's LLM provider, model, and **API key**. The
key is the only secret; it is stored via Anki's add-on config and never
transmitted anywhere except the configured LLM endpoint.

## Functional requirements

- **FR1 — Read-only guarantee.** The add-on MUST NOT call any note/card write
  API. No code path mutates the collection. This is verifiable by inspection:
  the add-on imports no `update_note` / `col.update_*` write surface.
- **FR2 — Single-card lint from the editor** on the in-progress note.
- **FR3 — Bulk lint from the browser** over all selected notes.
- **FR4 — Rubric coverage.** Every finding maps to a named principle from the
  rubric above; findings cite the principle by name.
- **FR5 — Actionable findings.** Each finding states the problem *in this card*
  and a concrete remediation the human can apply; no generic boilerplate.
- **FR6 — BYOK.** The LLM key/provider/model are user-supplied via config; the
  add-on ships with no key and no default endpoint that bills anyone but the
  user.
- **FR7 — Non-blocking UI.** LLM calls run off the Qt main thread; the UI stays
  responsive and shows progress for batch runs.
- **FR8 — Graceful failure.** Missing key, network error, rate limit, or
  malformed LLM output produce a clear message, never a crash and never a
  partial silent result. A batch run reports per-card failures without aborting
  the whole batch.
- **FR9 — No data exfiltration.** Card content is sent only to the user's
  configured LLM endpoint. No analytics, no third-party calls.

## Non-functional requirements

- **Privacy:** card content leaves the machine only to the user's chosen LLM.
- **Cost transparency:** batch runs warn before sending N cards (N × tokens is
  the user's money). Show the count and let them cancel.
- **Determinism of mapping:** the same card should map to the same set of
  principle violations across runs (low temperature; the rubric is fixed).
- **Offline-safe install:** the add-on does nothing on startup; no network call
  until the user explicitly lints.

## Technical approach (informative)

Not binding on implementation, but establishes feasibility. All symbols verified
against Anki 26.05 (Qt6, `addons21`, Python 3.9+).

### Integration surface

| Concern | Anki API |
|---------|----------|
| Editor button | `gui_hooks.editor_did_init_buttons` + `editor.addButton(...)`; callback fires after note-save so `editor.note` is valid |
| Browser bulk action | `gui_hooks.browser_will_show_context_menu`; `browser.selected_notes()` |
| Read fields | `note.items()` → `(field_name, value)` pairs; `note.note_type()` for context |
| Load note by id (batch) | `mw.col.get_note(nid)` |
| Config + key storage | `mw.addonManager.getConfig(__name__)` / `writeConfig`; `setConfigAction` for a custom config GUI; ship `config.json` defaults + `config.md` docs |
| Results dialog | `aqt.utils.showText(..., copyBtn=True)` or a custom `QDialog`; **no** `col.update_note` call anywhere |
| Background LLM call | `QueryOp(parent, op, success).without_collection().with_progress(...).run_in_background()` |

The read-only guarantee (FR1) falls out of the architecture: the only collection
API the add-on touches is `get_note` / `note.items()` (both reads). There is no
write call to remove or audit — it simply never exists in the codebase.

### Data flow

```
editor.note / get_note(nid)        ← read fields (main thread)
        │
   build rubric prompt             ← fixed rubric + card fields
        │
   QueryOp.op (background thread)   ← POST to user's LLM endpoint w/ user's key
        │
   parse findings → success(...)    ← back on main thread
        │
   showText(findings, copyBtn=True) ← read-only panel; nothing written back
```

UI data (selected note ids, field values) is collected on the main thread
*before* `run_in_background()`; the background op touches no Qt and no
collection lock.

### Packaging

Standard `.ankiaddon`: `__init__.py`, `config.json` (default `{"api_key": ""}`),
`config.md`, `manifest.json` (`package`, `name`, `min_point_version` in
`int_version` form), `user_files/` for anything that must survive upgrades.

## Acceptance criteria

The product is "done" for v1 when:

1. A card written in the editor can be linted with one click and produces
   per-principle findings within a few seconds.
2. Selecting N cards in the browser and choosing "Lint selected" produces a
   per-card report; one card's failure does not abort the batch.
3. Every finding names a rubric principle and gives a card-specific reason — a
   reviewer can trace each finding back to Matuschak or Wozniak.
4. A deliberately over-stuffed card (e.g. the Dead Sea "complex and wordy"
   example from Wozniak) is flagged for the Minimum Information Principle; a
   well-formed atomic card returns "looks good."
5. With no API key configured, linting shows a clear "configure your key"
   message and no crash.
6. Inspection confirms no collection-write API is reachable from any code path.

## Validation

- **Golden-set test:** a small fixture deck of known-good and known-bad cards
  (drawn from the worked examples in both essays — e.g. Wozniak's Dead Sea
  before/after, EU-membership set → enumeration). Assert bad cards trigger the
  expected principle and good cards stay clean. This is the regression harness;
  it pins the rubric mapping, not the LLM's exact prose.
- **Read-only assertion:** a test (or static check) that the add-on module
  references no write API.
- **Failure-path checks:** missing key, network error, and malformed LLM
  response each surface a clean message.

## Risks & open questions

- **LLM judgment variance.** The rubric mapping must be stable enough to trust.
  Mitigation: fixed rubric in the system prompt, low temperature, golden-set
  regression. Open question: do we pin a recommended model, or stay
  provider-agnostic and accept variance?
- **Cost on large decks.** Linting a 5,000-card deck is real money. Mitigation:
  explicit pre-send count + confirm; encourage selection-based runs over
  whole-deck runs.
- **Card formats beyond Front/Back.** Cloze, image-occlusion, and rich-HTML
  notes need sensible field extraction. Open question: how much HTML/markup do
  we strip before sending, and do we special-case cloze cards (where the deletion
  *is* the structure)?
- **Where suggestions stop.** The line "suggest, never apply" is the product's
  spine. A suggestion that quotes a fully-rewritten card edges toward authoring.
  Open question: do we cap suggestions at "what to change," not "here's the
  finished card"?

## Future work (post-v1)

- User-defined or weighted rubric rules (custom checks beyond the two essays).
- Inline severity hints in the editor as you type (debounced), not just on
  demand.
- A whole-deck "health report" summarizing the most common violations across a
  deck, to guide where to spend authoring effort.
- Cloze-aware critique that understands overlapping deletions.
