# Selling the GearboxSim Visualizer to Siemens Advanta — Pricing & Strategy

**Context:** After the steering-committee presentation, Siemens Advanta Consulting expressed interest in the GearboxSim Visualizer tool. This note covers an honest price to ask as a student, the best overall strategy, and the working-student / master's-thesis angle. Location: Munich, Germany.

> Caveat: euro figures are well-reasoned market estimates for the Munich 2026 market, not live quotes. Verify against current postings and the actual practicum/IP terms before committing.

---

## Bottom line first

**Don't lead by selling the tool. Sell *you*, with the tool as the door-opener.**

A one-off asset sale is the worst way to capture the value here. The tool's worth isn't the ~4,600 lines of code — it's the rare combination of *PlantSim domain knowledge + the dev skill to extend it*, and the fact that the tool is mid-transformation (the dynamic-topology rework isn't finished). A buyer who takes the code cheaply and loses the author got the bad half of the deal. **Tie the two together.**

**Target outcome, in priority order:**

1. Werkstudent role (continue developing it in-house)
2. Master's thesis with them (academic credit + they fund productionization)
3. Only if neither: a licensed / paid handover

---

## What is honestly being sold (and its limits)

Being clear-eyed keeps the pitch credible.

**Strengths**

- Instant bottleneck diagnosis from a raw PlantSim HTML export
- Zero-install single file, runs in any browser
- Genuinely accelerated the V1 → V2 iteration loop

**Honest gaps**

- Niche (PlantSim-only)
- Single developer, no test suite, no support / SLA
- Topology only just becoming model-agnostic

→ It is an **internal accelerator, not a product**. Price it as such — that's fine, because the value is *accelerator + the person who can grow it*.

---

## Settle these THREE things BEFORE discussing any number

1. **IP / ownership.** Built on a personal repo, in personal time → it's the author's. But check the HM München × Siemens Advanta practicum terms for any IP-assignment clause. If coursework IP stays with the student (the usual case), it's free to license or sell.
2. **Confidential data.** The tool is generic, but the sample files (`results_*.htm`, the deck) carry "Restricted | © Siemens Advanta" Gearbox Inc. data. **Strip all of it** before any demo or handover — ship the empty tool only.
3. **Exclusive vs non-exclusive.** Non-exclusive (they use it, the author keeps the engine for portfolio / future work) is better and allows a lower, clear-conscience price. Exclusive / buyout means giving up reuse — price materially higher.

---

## The numbers (Munich, 2026 — honest ranges)

| Path                                                         | What to ask                                                                                                   | Notes                                                                                                                                                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Werkstudent** (best)                                 | **€18–€22/h**, ~20 h/week                                                                            | ~€1,400–€1,900/mo gross. Keep <20 h/week to retain the Werkstudentenprivileg (reduced social-security). The real prize: income + Siemens on the CV + network + likely full-time offer. |
| **Master's thesis** with them                          | Werkstudent contract alongside, or stipend**€1,000–€1,500/mo**                                       | Thesis topic almost writes itself:*"Interactive material-flow & bottleneck-analysis layer for Tecnomatix Plant Simulation."*                                                            |
| **Freelance to productionize**                         | **€400–€600/day**, or fixed package **€4k–€8k** for a defined deliverable + handover + docs | If they want it built out but can't hire yet. Note Kleinunternehmer §19 UStG when invoicing.                                                                                             |
| **Non-exclusive license** (as-is)                      | **€2,000–€5,000** one-time                                                                           | Author keeps the engine. Reasonable if they just want today's version.                                                                                                                    |
| **Exclusive buyout** (incl. handover + support window) | **€8,000–€15,000**                                                                                   | Justified by time to make it production-grade + transfer + surrendering reuse.**Never below ~€2k for any sale** — a trivial number signals trivial work.                          |

---

## Recommended strategy / sequence

1. **Don't quote a price first.** Reply with enthusiasm + a question: *"Great to hear — are you thinking about using the tool internally, having me develop it further, or as a thesis collaboration? I'd love to explore the best fit."* Let them reveal the shape and the budget.
2. **Anchor on value, not code.** "It collapsed our simulation-analysis loop from minutes to seconds and drove the V1→V2 iteration. I'm already extending it to be fully model-agnostic with an automatic topology export from PlantSim." (Showing roadmap = the author is the asset.)
3. **Offer a tiered proposal** (Werkstudent / thesis / license) so they choose — every branch keeps the author involved.
4. **Keep optionality.** Prefer a non-exclusive license unless the buyout number genuinely compensates for losing reuse.
5. **Get it in writing** — even a one-page agreement. For employment, a standard Werkstudent contract covers it.

---

## The reframe for the conversation

The honest pitch isn't *"buy my software."* It's:

> *"You've seen what one student built in a practicum to make PlantSim analysis faster. Imagine that maintained and extended inside your team — and I'd like to be the person who does it."*

That is worth far more to Siemens Advanta than a €5k code drop — and it's the more honest value, too.

---

## Technical talking points (integration & scaling)

Siemens' real interest is **very large models (~1000 machines)**, where reading tables is exponentially more tedious. Use these points — they show you understand the platform's limits, which is what makes you hireable. (Full detail in `documentation.md` § *Integration & Scaling Notes*.)

- **The value scales, the current build does not — say so.** Reading tables is O(n) tedium; the ranked-constraint view is ~O(1) ("show me the 5 red nodes among 1000"). So the tool is *more* valuable at 1000 machines — **if** it renders. The current **SVG** renderer tops out in the low hundreds of nodes.
- **What scaling needs (a project, not a tweak):** swap SVG → **Canvas/WebGL**; mirror PlantSim's **nested Frame hierarchy** with drill-down/semantic zoom; index the parser (`Map`, not linear scans); make the **diagnostic top-N view the primary surface**; virtualize the station table.
- **Integration depth, honestly framed:** *auto-export + auto-launch* (days) and a *COM companion app* (1–3 weeks) are things Advanta can adopt now; an *embedded panel* is medium and hinges on PlantSim's web-control engine (WebView2 vs legacy IE — verify first); a *native product feature* belongs to the **Plant Simulation product team at Siemens DI Software**, not Advanta.
- **Why this argues for hiring you, not buying code:** the 1000-machine version is a WebGL + hierarchy rewrite. A static code drop doesn't get them there; the author who knows the domain and the codebase does.
- **Thesis framing:** *"Scaling an interactive factory-simulation analysis layer to 1000+ machines via WebGL rendering and hierarchical semantic zoom."*

---

## Practical checklist

- [ ] Confirm IP terms in the practicum agreement
- [ ] Strip all Siemens / Gearbox Inc. confidential data from the tool and samples
- [ ] Decide preferred outcome (Werkstudent first) and walk-away minimums
- [ ] Reply to their contact with the open question (don't quote a price)
- [ ] Prepare the tiered one-pager for internal circulation
- [ ] Get the final arrangement in writing

---

*Authored 2026-06-19. Companion to the GearboxSim Visualizer (`gearboxsim.html`) and the steering-committee materials.*
