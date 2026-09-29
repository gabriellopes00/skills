# Architecture Document Reference

How to produce the architecture write-up in Phase 7. The deliverable is a document a new engineer can read start-to-finish and then navigate the codebase — not a slide deck and not a diagram dump.

The original source skill shipped a branded HTML template with a fixed palette. That asset is intentionally **not** bundled here, because it tied the output to one visual identity. What follows is the part that is portable: **which sections to write, which diagrams earn their place, and how to draw them consistently** — in whatever format the project uses (Markdown in the repo, an HTML page, a wiki, an ADR set).

## Table of Contents

1. Choosing the format
2. Standard sections
3. Diagram catalog
4. Diagram conventions
5. Consistency and accessibility
6. Keeping it alive

---

## 1. Choosing the format

| Format | Use when |
| --- | --- |
| **Markdown in the repo** (`docs/architecture.md`) | Default. Versioned with the code, reviewable in PRs, diffable. Use Mermaid for diagrams so they render on the host and stay editable. |
| **Single self-contained HTML file** | You need precise hand-drawn-style SVG diagrams, print fidelity, or distribution to non-engineers. Opens in a browser with no build step. Inline all CSS; embed all assets. |
| **ADR set** (`docs/adr/NNNN-*.md`) | Recording *decisions* over time rather than describing the current state. Complements the main document; does not replace it. |
| **Wiki / Confluence** | The organization mandates it. Accept that it will drift from the code — link to the repo document as the source of truth. |

Keep the content **vendor-neutral** unless the user asks for branding. Do not hard-code a company name into a reusable document.

---

## 2. Standard sections

Order them consistently, and give each one focused diagram **where a diagram earns its place**. A section with nothing visual to show is better as prose than as a decorative box-and-arrow.

1. **Overview** — what the platform is, in three sentences. Who uses it, what problem it solves, what shape it is (one deploy / several).
2. **Domains (DDD)** — the context map: Core / Supporting / Generic, plus external systems behind the ACL.
3. **Principles** — the modular and structural principles this system commits to, and any deliberate exceptions.
4. **Module map** — the physical layout: composition roots, module libraries, shared.
5. **Bounded contexts** — per context: responsibility, ubiquitous language, key aggregates, public contract.
6. **Anti-Corruption Layer** — the ports and their adapters; which vendor sits behind each.
7. **Communication** — sync vs async decisions; the event catalog; the transactional outbox.
8. **Front-to-back** (if there is a client) — the contract flow: schema → generated client → transport → push channel.
9. **Module internals** — the flat-by-aggregate layout and how the dependency rule is preserved.
10. **Workflow** — the durable pipeline, if there is one.
11. **Data model** — the ERD, grouped by owning context, showing in-context relations vs cross-context id references.
12. **Resilience** — the timeout → breaker → retry policy and the fallbacks.
13. **Real-time** — the push channel and its fan-out.
14. **Evolution** — the staged granularity plan and the trigger for each stage.
15. **Stack** — the technology table, each row with its rationale.

Not every system needs all fifteen. Drop the sections that do not apply (a worker-only system has no front-to-back and no real-time) rather than filling them with "N/A".

---

## 3. Diagram catalog

Reusable diagram types, all in one visual language:

- **Context map** — zones for Core / Supporting / Generic, plus a band for external systems behind the ACL.
- **Module map** — stacked layers: composition roots (apps) → module libraries (contexts) → shared.
- **Rich aggregate** — a root box listing its methods in the ubiquitous language, a child entity, value-object pills, and an invariant note.
- **Ports & Adapters** — four columns: core · port · adapter · external. Arrows for "uses", "implements" (dashed, pointing inward), and "calls".
- **Transactional outbox** — the business write and the outbox insert inside one atomic box → relay → bus → idempotent consumers.
- **Front-to-back** — a client box and a backend box, with request/response arrows and push arrows, plus the schema→client codegen band.
- **Flat-by-aggregate module** — a folder tree in monospace beside the dependency rule (controller → service → entity ← repository).
- **Durable pipeline** — a horizontal step pipeline under an orchestrator band.
- **ERD by context** — table boxes grouped and colored by owning context; **solid** lines for in-context foreign keys, **dashed** for cross-context id references. This one diagram makes the state-isolation rule visible at a glance.
- **Resilience layers** — nested rectangles (timeout ⊃ breaker ⊃ retry ⊃ call) plus a fallback box and the jitter formula.
- **Real-time fan-out** — clients ↔ instances ↔ pub/sub ↔ event source.
- **Sequence** — lanes with lifelines and numbered messages; a distinct arrow color per actor.
- **Evolution timeline** — staged boxes connected by arrows, with a "you are here" marker.

---

## 4. Diagram conventions

**With Mermaid** (Markdown, and the pragmatic default): use `flowchart` for context maps and module maps, `sequenceDiagram` for flows, `erDiagram` for the data model, `stateDiagram` for lifecycles. Keep node labels short; put the explanation in the caption, not inside the box. Style by role with `classDef` so colors mean the same thing across every diagram.

**With hand-authored SVG** (when you need precise control):

- Fixed **viewBox width** matching the content column (e.g. `viewBox="0 0 880 H"`), height per diagram. Always set `role="img"` and a descriptive `aria-label`.
- **Arrows:** one `<marker>` per diagram with a unique id, reused via `marker-end`. Color the arrow by meaning — neutral gray for flow, one accent for push/UI, another for domain/events, green for success/data, tan/gold for external calls.
- **Boxes:** rounded `rect` (`rx` 8–14). White fill with a colored stroke for items; a soft fill or light gradient for zones and bands.
- **Zones:** large soft-filled rounded rects with an uppercase, letter-spaced label in the top-left corner.
- **Text:** a clean sans-serif for titles (bold) and labels; a monospace face for identifiers, file paths, and folder trees. Keep secondary text around 9.5–10.5px in a muted gray.
- **Escape** `<`, `>`, and `&` inside SVG text as `&lt;`, `&gt;`, `&amp;`.

**In both cases:**

- **Number every diagram contiguously** and caption it: *"Diagram N — the ports and adapters around the ERP integration."* If you insert one, renumber the rest.
- Choose one color per architectural role and never reuse it for something else. A workable assignment: primary/Core · a second hue for supporting and apps · a third for domain and events · green for infrastructure · sand/tan for external systems.
- A diagram should show a **mechanism**, not a decoration. If the reader learns nothing from it that the prose did not already say, delete it.

---

## 5. Consistency and accessibility

- **One visual language** across all diagrams: the same palette, fonts, corner radius, and arrow style.
- Keep the navigation, the table of contents, the section anchors, and the diagram numbers in sync.
- Every diagram needs a text alternative — `role="img"` plus `aria-label` in SVG, or the caption plus surrounding prose in Markdown.
- Provide a **legend** whenever color carries meaning. Never let color be the *only* carrier: pair it with a label, a line style (solid vs dashed), or a shape.
- Wide content (tables, folder trees, wide diagrams) must scroll inside its own container rather than forcing the page to scroll sideways.
- If the document is rendered in a theme-aware host, define colors so they hold up in both light and dark.

---

## 6. Keeping it alive

An architecture document that describes last year's system is worse than none — people trust it and are wrong.

- **Store it next to the code** and review it in the same PR that changes the structure.
- **Link, don't duplicate.** Point at the actual module paths, event names, and port interfaces rather than re-describing them. When they are renamed, the broken link is the signal.
- **Record trigger conditions, not just decisions.** "We will split the reporting module when its p99 exceeds X" is checkable; "we may split reporting later" is not.
- **Log deviations.** When a module deliberately breaks a principle, write down which one, why, and what would have to change to fix it. An undocumented exception becomes precedent within a quarter.
- **Date the stack table.** Technology rows age fastest.
