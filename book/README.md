# Book — Software Architecture in Practice (Bass, Clements, Kazman)

Reference textbook for the course. Use these chapters when you need authoritative definitions of quality attributes, tactics catalogues, or QAS structure. Prefer the chapter PDF over the full book PDF — they cover the same material but the chapter file is far smaller.

## Files

| File | Topic | When to consult |
|---|---|---|
| `SA_book.pdf` | Full book (14 MB) | Only as a last resort — use chapter files instead. |
| `SA_book_ch4_QAS.pdf` | Understanding Quality Attributes | Definition of a Quality Attribute Scenario (source, stimulus, artifact, environment, response, response measure). Use when writing or refining any QAS. |
| `SA_book_ch5_availability.pdf` | Availability | Fault detection / recovery / prevention tactics. **Relevant to teammate's Av2 scenario** (Ping/Echo, heartbeat, exception detection). Don't edit Av2 work, but read this if you need to understand the seam. |
| `SA_book_ch6_interoperability.pdf` | Interoperability | Tactics for locating and managing interfaces (orchestrate, tailor). Useful for HIS integration discussions. |
| `SA_book_ch7_modifiability.pdf` | Modifiability | Tactics: reduce size of module, increase cohesion, reduce coupling, defer binding. Background for the initial M2 (New Clinical Models) scenario. |
| `SA_book_ch8_performance.pdf` | Performance | **Primary reference for P1 work.** Tactics: control resource demand (manage sampling rate, limit event response, prioritize events, reduce overhead, bound execution times, increase resource efficiency) and manage resources (increase resources, increase concurrency, maintain multiple copies, bound queue sizes, schedule resources). |
| `SA_book_ch9_security.pdf` | Security | Tactics: detect, resist, react, recover from attacks. Background only — no security QAS in scope for Part 2. |
| `SA_book_ch10-testability.pdf` | Testability | Background only. |
| `SA_book_ch11_usability.pdf` | Usability | Background only. |
| `SA_book_ch12_other_QAs.pdf` | Other Quality Attributes | Catalogue of additional QAs (e.g., safety, energy efficiency). Skim if a QAS doesn't fit the main seven. |
| `SA_book_ch16.pdf` | Architecture Evaluation (ATAM) | Source for the ATAM-inspired reasoning model used in CLAUDE.md §1.1 — sensitivity points, trade-off points, risks, non-risks. Read when justifying or critiquing an architectural decision. |

## Quick map: Part 2 QAS → chapter

- **P1 (Performance)** — my part → `ch8_performance.pdf` (resource demand + resource management tactics)
- **Av2 (Availability)** — teammate's part → `ch5_availability.pdf` (fault detection tactics: Ping/Echo, heartbeat)
- **ATAM reasoning** (sensitivity & trade-off points) → `ch16.pdf`
- **QAS structure** itself → `ch4_QAS.pdf`
