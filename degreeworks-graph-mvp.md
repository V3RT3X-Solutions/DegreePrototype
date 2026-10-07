# Project Brief: Interactive Degree Graph (Personal MVP)

Context file for Claude Code. Read it fully before writing code. Ask the user for the input file in Step 0 before building anything.

---

## 1. Goal

A small local web page that shows ONE student's degree as an interactive graph (nodes and edges), built from a single DegreeWorks audit the student exported by hand. This is a demo of the core idea (see your whole degree as a map), not the full product.

Works on one person's data only. No accounts, no backend, no deployment.

---

## 2. Hard constraints (do not violate)

- **No automation against any university system.** Do not scrape DegreeWorks or Athena, do not drive a browser against them, do not read network traffic from them, do not build a browser extension. The user exports their own audit manually (Save as PDF, or select-all and copy the text) and gives you the file.
- **No credentials.** Never ask for, store, or handle a UGA login.
- **Everything runs locally.** No network calls with the user's data. No analytics, no CDN calls that include data. Libraries may load from a local `vendor/` folder or npm, but the data never leaves the machine.
- **Strip personal identifiers during parsing.** Discard name, student ID number, GPA, and advisor name. Keep only course codes, course titles, credit hours, status, and requirement block names. Do not write the raw audit text into any output file.
- **Privacy hygiene.** Add `raw/` and `data/graph.json` to `.gitignore` before the first commit. If the user later wants to show the graph publicly, offer the redaction toggle in Step 5.
- **No LLM in the logic.** Status, eligibility, and highlighting are computed by plain code. Do not have a model decide what is satisfied.

---

## 3. What the audit does and does not contain

Important for scoping, so do not promise more than the data supports:

- A DegreeWorks audit lists requirement blocks (core, major, electives, etc.), the courses that satisfy each, and a status per course (complete, in progress, still needed).
- It does **not** reliably contain prerequisite chains for courses not yet taken. Those live in the UGA Bulletin.
- So the graph has two layers:
  1. **Requirement layer (from the audit):** Degree, then blocks, then courses, with status.
  2. **Prerequisite layer (optional, hand-entered):** a small `data/prereqs.json` the user fills in for the handful of courses they care about, copied by hand from the Bulletin. Prerequisite edges are drawn only where this file has entries.

Build layer 1 first. Layer 2 is a bonus once layer 1 works.

---

## 4. Data format

`data/graph.json` (generated, git-ignored):

```json
{
  "nodes": [
    { "id": "DEGREE", "type": "degree", "label": "B.S. Major Name" },
    { "id": "BLOCK:major-core", "type": "block", "label": "Major Core", "status": "in_progress" },
    { "id": "CSCI 1301", "type": "course", "label": "Intro to Programming", "credits": 4, "status": "complete" }
  ],
  "edges": [
    { "source": "DEGREE", "target": "BLOCK:major-core", "kind": "contains" },
    { "source": "BLOCK:major-core", "target": "CSCI 1301", "kind": "contains" },
    { "source": "CSCI 1301", "target": "CSCI 1302", "kind": "prereq" }
  ]
}
```

`status` is one of `complete`, `in_progress`, `needed`. A block's status is derived in code: all children complete means `complete`, any child in progress or complete but not all means `in_progress`, none started means `needed`.

`data/prereqs.json` (hand-edited, optional):

```json
{ "CSCI 1302": ["CSCI 1301"], "CSCI 2720": ["CSCI 1302", "MATH 2200"] }
```

The key is the course, the list holds its prerequisites. Courses mentioned here that are not in the audit appear as gray "not in audit" nodes.

---

## 5. Stack

- **Parser:** Python 3 script, `parse_audit.py`. Use `pdfplumber` or `pypdf` for PDF input, or plain text if the user pastes text. Output `data/graph.json`.
- **Viewer:** one static `index.html` plus `app.js`, using **Cytoscape.js** for the graph. No build step, no framework. Open with a tiny local server (`python -m http.server`) so the page can fetch `graph.json`.
- **Layout:** Cytoscape's `dagre` layout (left to right) for the prerequisite view, `cose` or `breadthfirst` for the requirement tree. Let the user toggle.

---

## 6. Repo structure

```
degree-graph/
  raw/                 # user's exported audit goes here (git-ignored)
  data/
    graph.json         # generated (git-ignored)
    prereqs.json       # hand-edited
  parse_audit.py
  index.html
  app.js
  style.css
  README.md
  .gitignore
```

---

## 7. Build steps (finish and verify each before the next)

- [ ] **Step 0: Get the input.** Ask the user to export their own audit (Save as PDF from the DegreeWorks page, or copy the text) and drop it in `raw/`. Open it and look at the real layout before writing any parsing code. Show the user a short summary of what you detected (block names, number of courses, statuses) and confirm it is right.
- [ ] **Step 1: Parser.** Write `parse_audit.py` against the user's actual file. Extract blocks, courses, credits, and status. Drop all personal identifiers. Print a readable summary and write `data/graph.json`. If some lines can't be parsed confidently, list them for the user rather than guessing. A hand-corrected `graph.json` is an acceptable fallback.
- [ ] **Step 2: Static graph.** `index.html` loads `graph.json` and renders it with Cytoscape. Color by status: complete = green, in progress = amber, needed = gray with a strong outline. Node shape differs by type (degree, block, course).
- [ ] **Step 3: Interactivity.**
  - Click a node: side panel shows code, title, credits, status, and which block(s) it counts toward.
  - Click a course: highlight everything upstream (its prerequisites) and downstream (what it unlocks) from `prereqs.json`, dim the rest.
  - Filter buttons: show all, only needed, only in progress.
  - Search box: type a course code and jump to it.
  - Layout toggle: requirement tree vs prerequisite flow.
- [ ] **Step 4: Prerequisite layer.** Load `prereqs.json`, add `prereq` edges (dashed, arrowheaded), and add an "eligible now" highlight: a needed course whose prerequisites are all complete gets a distinct border.
- [ ] **Step 5: Demo polish.**
  - Progress summary at the top ("X of Y credits complete").
  - A **redact toggle** that replaces course titles with codes only, for screenshots or showing others.
  - Clean, high-contrast styling that reads on a projector (large labels, dark or light theme, no tiny text).
  - Export the current view as a PNG for slides.
- [ ] **Step 6: README.** How to export the audit, run the parser, start the local server, and edit `prereqs.json`.

---

## 8. Not in scope

- Any live connection to DegreeWorks, Athena, or UGA systems
- Schedule generation or the constraint solver
- Seat monitoring, notifications, or registration of any kind
- Multiple students, accounts, or hosting
- LLM or conversational features
- Neo4j or any database

---

## 9. Definition of done

Running the parser on the user's exported audit produces `graph.json`, and opening `index.html` through a local server shows their degree as a colored, clickable graph where they can see what is complete, what is left, what is eligible next, and what each course unlocks. The whole thing can be screenshotted or screen-recorded for a pitch slide, with a redacted mode available.
