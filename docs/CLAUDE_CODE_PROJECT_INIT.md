# SAMI / Southbrook Cabinetry — Claude Code Project Initialization

> **Paste this entire file into Claude Code as the opening instruction, or commit it as `CLAUDE.md` at the repository root.**
> It is the single source of truth for how this repository is built. Read it fully before writing any code.

---

## 0. Mission

Build the **Southbrook Cabinetry AI Kitchen Platform** on **Odoo v19 Community Edition** — a photo-to-factory digital thread that takes a customer kitchen photo, generates design concepts, places cabinets via a rules engine, produces parametric CAD and manufacturing artefacts via FreeCAD, controls revisions via a custom PLM module, and releases jobs to MRP. Odoo is the system of record. FreeCAD is the engineering engine. Gemini is the room-intelligence engine. Flutter is the customer experience layer.

You will generate the complete monorepo: Odoo addon modules (models, security, views, services, tests), the FreeCAD bridge microservice, the FreeCAD parametric template library, and the Docker environment. **Build incrementally, one module at a time, with an explicit acknowledgment gate after every module.**

---

## 1. Operating Disciplines (NON-NEGOTIABLE)

These are carried forward from the established project working agreement. Apply them on every commit.

- **Discipline A — Regression first.** Before adding any new method to an existing model/service, confirm the existing methods still have passing tests. Never add behaviour on top of untested behaviour.
- **Discipline B — Smoke-test stub promotion.** Every stub you create gets a smoke test that fails until the stub is implemented. No silent placeholders.
- **Discipline C — Autonomous-stretch review cadence.** Work autonomously within a single module, but STOP at each module boundary and request acknowledgment before starting the next module. Do not chain modules without a gate.
- **Discipline D — Disk verification precedence.** Before asserting that any code, field, or behaviour exists or is broken, verify it **on disk** (read the file, run the test, query the DB). Disk truth beats memory and beats pasted snippets. If you cannot verify on disk, say so explicitly and ask.

**Commit format:** Conventional Commits. Body must cite the decision/requirement identifier it satisfies (e.g. `Refs: D-FC-02, GAP-04`). One logical change per commit. Tests in the same commit as the code they cover.

**Acknowledgment gate protocol:** At the end of each module, output (a) what was built, (b) test results (pass/fail counts), (c) what the next module is, (d) any blocking question. Then STOP and wait for explicit "proceed" before the next module.

---

## 2. First Actions — Inspect Before You Build

**Do these in order before generating anything. This is Discipline D applied to project bootstrap.**

1. **Inspect the live instance & existing code.** The production Odoo instance is at `https://southbrookcabinetry.space`. The existing addons must be read from disk in the repository / Odoo addons path before you touch them. Identify and read the manifest, models, and tests of:
   - `southbrook_estimating` — **already deployed: 16 commits, ~6,300 lines, 95 test methods, 27 Q/NF decision identifiers.** It owns attribute configuration, variant creation, pricelists, cabinet templates, geometric conventions, and the **7-routine boundary** (do not exceed 7 custom routines in this addon).
   - Any in-progress `southbrook_customer_portal` work (Phase 1.6 OWL + JSON-RPC `/my/order-builder`).
   - The OCA `product_configurator` trio (`product_configurator`, `_sale`, `_mrp`) and `website_product_configurator` as currently vendored.

2. **Report the actual state.** Output a short inventory: which modules exist, their test counts, their model lists, and any divergence from what this document assumes. **If reality differs from this document, reality wins — flag the divergence and ask before proceeding.**

3. **Confirm the environment.**
   - Odoo 19.0 CE + PostgreSQL 16, Docker-hosted.
   - Confirm a disposable local Odoo 19 clone is available for test runs (NOT the QNAP AlfaCore stack at `192.168.68.108`, which is the deployment target, not the validation target).
   - Confirm Python 3.11+, Node.js (for OWL assets), and the docx/openpyxl toolchain.

4. **Do NOT rebuild `southbrook_estimating`.** It is the foundation. You extend it via `_inherit` from other modules. The only changes permitted to it are additive and must respect the 7-routine boundary.

---

## 3. Hard Constraints

- **CE-only.** No Enterprise modules. PLM, Quality, Field Service, Sign, Studio, Shop Floor tablet, Barcode, and full Accounting are Enterprise in v19. Every capability that would use one of these must map to an OCA module, a custom module, or an explicit deferral. State the mapping in the module manifest comments.
- **`create_get_bom` is the BoM authority.** Static `mrp.bom.line` quantities cannot encode parametric area-based quantities. Per-variant exact quantities are resolved at sale time by `product_configurator_mrp`'s `create_get_bom`. Never hardcode panel quantities in a static BoM.
- **BoM-contents assertions are a deployment gate.** No FreeCAD overlay or factory deployment of any module that produces a BoM until `test_bom_contents.py` asserts the actual contents of `create_get_bom` output against the canonical panel formulas. This blocks Phase B onward. It is the single most important safety gate in the project.
- **Odoo 19 API specifics — apply throughout:**
  - `safe_eval`: the `globals_dict=` argument was renamed to `context=`.
  - `self.env.context` is no longer mutable — use `self.with_context(ctx)` instead of `self.env.context = ctx`.
  - Manufacture route external ID is `mrp.route_warehouse0_manufacture` (real ID, not a placeholder).
- **ACL testing discipline.** Admin-run tours cannot surface public-buyer ACL defects. Any portal/website feature must be tested as a genuine anonymous second customer. The OCA port already has three documented latent defects (public-buyer ACL on option-product reads; reconfigure-route ACL/IDOR; JSON int-vs-string session key bug) — do not reintroduce these patterns.
- **Configured-product model (locked).** Finite cosmetic choices (door style, finish, decor) are dynamically-created variants. W×H×D are numeric parametric inputs that drive the formula BoM — **never** modelled as product attributes.

---

## 4. Repository Structure

Generate this monorepo layout. Create directories lazily as each module is built, not all upfront.

```
southbrook-platform/
├── CLAUDE.md                          # this file
├── docker-compose.yml                 # Odoo + Postgres + FreeCAD Bridge
├── .env.example                       # secrets template (never commit real secrets)
├── README.md
│
├── addons/                            # Odoo CE addons
│   ├── southbrook_estimating/         # EXISTS — review only, extend via _inherit
│   ├── southbrook_plm/                # Module 1
│   ├── southbrook_freecad_bridge/     # Module 2 (Odoo side of the bridge)
│   ├── southbrook_hardware_catalog/   # Module 3
│   ├── southbrook_kitchen_mrp/        # Module 4 (sb.cutlist, sb.hardware.package)
│   ├── southbrook_kitchen_workspace/  # Module 5 (sb.kitchen.project ...)
│   ├── southbrook_ai_design/          # Module 6 (Gemini caller, JSON contract)
│   ├── southbrook_config_engine/      # Module 7 (cabinet placement rules engine)
│   ├── southbrook_customer_portal/    # Module 8 (extends Phase 1.6 portal)
│   └── southbrook_dealer_portal/      # Module 9 (dealer 50% pricing, KD program)
│
├── services/
│   └── freecad_bridge/                # FastAPI microservice (the only FreeCAD dependency)
│       ├── Dockerfile
│       ├── requirements.txt
│       ├── main.py                    # /render /status /validate /templates /health
│       ├── worker.py                  # async job queue consumer
│       ├── freecad_runner.py          # subprocess wrapper for freecadcmd
│       ├── odoo_client.py             # XML-RPC: auth, create attachments, write status
│       └── scripts/
│           ├── render_cabinet.py      # runs inside freecadcmd
│           ├── export_dxf.py
│           ├── export_shopdrw.py
│           └── export_step.py
│
├── freecad_templates/                 # parametric .FCStd masters (one per cabinet family)
│   ├── base_cabinet.FCStd
│   ├── wall_cabinet.FCStd
│   ├── tall_cabinet.FCStd
│   ├── vanity_cabinet.FCStd
│   ├── drawer_unit.FCStd
│   └── corner_cabinet.FCStd
│
├── shared/
│   └── southbrook_dims.py / .js        # SINGLE SOURCE: panel formulas, used by FreeCAD + Three.js + assertions
│
├── flutter_app/                        # customer mobile app (built last)
│
└── docs/
    ├── api_contracts/                  # Flutter↔Odoo, Gemini↔Odoo, Bridge webhook
    ├── config_engine_spec.md           # the largest spec — written before Module 7
    └── ai_prompt_spec.md               # Gemini prompts — written before Module 6
```

---

## 5. Build Sequence & Per-Module Definition of Done

Build **strictly in this order**. Each module ends with a gate (Discipline C). Dependencies are explicit — do not start a module whose dependency has failing tests.

### Module 0 — Repository skeleton + Docker (no business logic)
- `docker-compose.yml`: `odoo` (19 CE), `db` (postgres:16), `freecad-bridge` (placeholder build).
- `.env.example` with `FREECAD_BRIDGE_SECRET`, `ODOO_DB`, `GEMINI_API_KEY` placeholders.
- `shared/southbrook_dims.py` — implement the 7 canonical panel formulas (Side, Top, Bottom, Back, Adjustable Shelf, Toe Kick, Door Face) as the authoritative dimension source for FreeCAD, Three.js, and the BoM-contents assertions. **Peter Tuschak signed off these formulas 2026-06-09 (G2 closed). Treat them as final. Do NOT add provisional / TODO / SIGN-OFF-REQUIRED markers.**
- **DoD:** `docker compose up` brings Odoo to the login page; `southbrook_estimating` loads without error; bridge container builds and answers `GET /health`.

### Module 1 — `southbrook_plm`
- **Depends on:** `mrp`, `product`, `mail`, `southbrook_estimating`.
- **Gate before starting:** run a 5-minute check on `github.com/OmniaSolutions/odoo-plm` for a stable 19.0 branch. If one exists and covers ECO + BoM revision, adopt it and layer FreeCAD fields on top instead of building from scratch. Report the decision.
- **Models:** `plm.document`, `plm.eco`, `plm.eco.line`, `plm.bom.revision`. Add `x_cad_status`, `x_cad_attachment_ids`, `x_plm_eco_id` to `mrp.production` via `_inherit`.
- **ECO state machine:** `draft → in_review → approved → done`, plus `rejected` and `cancelled`. Approval creates a `plm.bom.revision` snapshot, bumps the BoM version, and resets `x_cad_status` to `pending`.
- **Revision types** (from Master Spec §11): Concept, Design, Engineering, Production.
- **Security:** PLM Reviewer group; `ir.model.access.csv`; record rules so a customer never sees ECO internals.
- **Views:** ECO form with status bar, colour-coded `plm.eco.line` (removals red, additions green, modifications amber), CAD Files tab, Approvals tab, chatter.
- **Tests:** `test_plm_eco.py` (state transitions, snapshot creation, version bump). Smoke test for every stub.
- **DoD:** all tests green; ECO lifecycle demonstrable end-to-end on the local clone.

### Module 2 — `southbrook_freecad_bridge` (Odoo side) + the FreeCAD service
- **Depends on:** `southbrook_plm`, `mrp`.
- **GATE — CRITICAL (G1):** `test_bom_contents.py` must exist and pass for the base cabinet before this module deploys anything. It asserts `create_get_bom` output against `shared/southbrook_dims.py`. **Do not skip. Do not defer.** The 7 panel formulas are signed off (G2 closed 2026-06-09), so build the test harness against the **final** formulas — no provisional shim.
- **GATE — DEPLOYMENT HOLD (owner-confirmation):** Even after G1 turns green, **Module 2 remains deployment-blocked until the project owner (dangelo.john@gmail.com) gives explicit written go-ahead.** Build the module, run the tests, produce the acknowledgment report, then STOP. Do not POST to the bridge, do not enable the `mrp.production` confirm server action, and do not write the kanban CAD-status badges into a deployed view set until the owner confirms. This is a hard gate independent of Discipline C.
- **Odoo side:** server action on `mrp.production` confirm → POST job to bridge; `/plm/cad_callback` controller receives attachment IDs + status; shop-floor kanban CAD-status badges (grey/amber/green/red); CAD Artefacts tab; "Regenerate CAD" button (Manager group only).
- **Service side (`services/freecad_bridge`):** FastAPI with `/render`, `/status/{job_id}`, `/validate`, `/templates`, `/health`. Shared-secret header auth. Fire-and-forget async job pattern; XML-RPC back to Odoo for `ir.attachment` create and `mrp.production` status write. `render_cabinet.py` opens the FCStd template, writes the Spreadsheet B-column, recomputes, exports **DXF R12** (per panel), **SVG/PDF** shop drawing (TechDraw or SVG→cairosvg — prototype both per open question Q-02), **STEP AP214** assembly.
- **FreeCAD env gate:** confirm FreeCAD 1.0 AppImage exposes `freecadcmd` headless on the Docker base OS. If absent, choose an alternative packaging and report it before writing templates.
- **Tests:** Odoo controller test (`test_cad_callback.py`), bridge unit tests (job schema validation, secret auth), and 18 render smoke tests (6 templates × 3 sizes).
- **DoD:** confirming an MO triggers a render; artefacts land on the MO; `x_cad_status` reaches `done`; BoM-contents assertions green.

### Module 3 — `southbrook_hardware_catalog`
- **Depends on:** `product`, `purchase`.
- **Source asset:** the primary hardware-supplier workbook (179 products, 49 subcategories, ~19 brands). Import as `product.supplierinfo` + `product.brand`. Cost/price fields require trade-account login — mark unresolved SKUs `x_pricing_pending = True`.
- **Hardware resolution:** `hardware_map.json` mapping `{cabinet_type, door_count, drawer_count, shelf_count}` → `[{sku, qty}]`. Expose a method the bridge calls post-render to append hardware lines to the BoM.
- **Tests:** import integrity (179 rows), resolution lookup correctness, pending-pricing flag behaviour.
- **DoD:** a base cabinet variant resolves to a concrete hardware pick list (hinges, slides, pins) with correct quantities.

### Module 4 — `southbrook_kitchen_mrp`
- **Depends on:** `mrp`, `southbrook_hardware_catalog`, `southbrook_freecad_bridge`.
- **Models:** `sb.cutlist` (lines: panel_name, qty, width_mm, height_mm, substrate, grain_dir, edge_banding_config), `sb.hardware.package`, `sb.production.package`.
- **Cut list is first-class** (gap GAP-05): the bridge writes panel geometry to `sb.cutlist` lines; a nesting prep step (opticutter or custom) optimises before DXF export. Design the `sb.cutlist` ↔ nesting interface with the internal cutting/nesting division workflow in mind (SYN-02).
- **Tests:** cutlist generation from a BoM, hardware package assembly, nesting input/output round-trip stub.
- **DoD:** an MO produces a complete `sb.cutlist` + `sb.hardware.package`.

### Module 5 — `southbrook_kitchen_workspace`
- **Depends on:** `southbrook_estimating`, `crm`, `mail`.
- **Models:** `sb.kitchen.project`, `sb.kitchen.design.option`, `sb.kitchen.ai.analysis`, `sb.kitchen.appliance`, `sb.kitchen.approval`. **Specify every field** — the source docs only name the models.
- **UX:** 3-panel OWL workspace (Left: customer/opportunity/theme; Center: photos/AI analysis/design options; Right: CAD preview/revision history/approval status; Footer: Validate Layout / Generate CAD / Generate Quote / Submit for Approval / Release to Production).
- **Tests:** project lifecycle, design-option selection, approval record creation.
- **DoD:** a designer can create a project, attach photos, and select among design options in the workspace.

### Module 6 — `southbrook_ai_design`
- **Depends on:** `southbrook_kitchen_workspace`.
- **GATE before starting:** author `docs/ai_prompt_spec.md` AND `docs/api_contracts/gemini_odoo_contract.md` first (gaps GAP-03, GAP-08). The Gemini output schema MUST match the Configuration Engine input schema exactly.
- **Critical correctness rule (GAP-02):** Gemini provides appliance positions and approximate proportions — **not** mm-accurate dimensions. The JSON contract must carry a `confirmed_by_human: bool` flag on all dimensional fields. The Configuration Engine MUST refuse to run while any required dimension has `confirmed = false`. Build a manual dimension-confirmation step into the workspace.
- **Models/services:** Gemini API caller, JSON contract validator, prompt template storage, `sb.kitchen.ai.analysis` population.
- **Tests:** schema validation (reject malformed Gemini output), human-confirmation gate enforcement, graceful fallback (`sink_detected: false` not hallucinated).
- **DoD:** a photo produces a validated, schema-conformant analysis record with unconfirmed dimensions flagged.

### Module 7 — `southbrook_config_engine` (LARGEST — platform critical path)
- **Depends on:** `southbrook_kitchen_workspace`, `southbrook_estimating`.
- **GATE before starting:** author `docs/config_engine_spec.md` (gap GAP-01). This is the brain of the platform and currently has **zero business rules specified anywhere**. Model it on AYA Kitchens' cascade (Collection → Door → Material → Finish) and the Prodboard constraint-solver pattern (`PRODBOARD_MANIFEST.md`).
- **Scope:** cabinet placement rules, filler width calculation (run width − Σ cabinet widths), corner solutions (blind corner / lazy Susan / diagonal / 45° filler), appliance clearances (stove, fridge, dishwasher, sink), conflict resolution. Store rules as Odoo records (`sb.placement.rule`), not hardcoded.
- **Output:** a validated cabinet arrangement that FreeCAD can render as a full kitchen assembly (not just individual cabinets).
- **Tests:** galley / L-shape / U-shape / island / peninsula layouts; filler math; clearance enforcement; conflict handling.
- **DoD:** a confirmed room + appliance set produces a valid, manufacturable cabinet arrangement.

### Module 8 — `southbrook_customer_portal`
- **Depends on:** all of Modules 1–7; extends Phase 1.6 `/my/order-builder`.
- **Scope:** concept A/B/C review UI, customer approval workflow (wired to `sb.kitchen.approval` and the ECO state machine), quote PDF delivery, "shop drawings ready" email. **Phase 2:** Three.js `KitchenCanvas` OWL component rendering the Configuration Engine output live in the browser (geometry from `shared/southbrook_dims`, NOT a served STEP file — D-FC-06).
- **Tests:** ACL as anonymous second customer (mandatory), approval flow, Three.js scene parity with FreeCAD dimensions (SYN-03 / R-07 single-source check).
- **DoD:** a customer reviews 3 concepts, approves one, and receives a quote + drawings via the portal.

### Module 9 — `southbrook_dealer_portal`
- **Depends on:** `southbrook_customer_portal`, `southbrook_plm`.
- **Scope:** dealer-specific pricing view (contractual 50% off retail), dealer order entry, installation-drawing PDF delivery (gap GAP-06 — separate FreeCAD TechDraw elevation output), and a KD flat-pack export variant for the Central Kitchens channel (SYN-05: pre-drilled hardware hole positions in `sb.cutlist`).
- **Tests:** dealer pricing correctness, installation-drawing generation, KD export variant.
- **DoD:** a dealer (one of the four contracted dealers) runs photo → concept → 50% quote → approve → manufacturing → installation PDF, fully automated.

### Flutter app (built last, parallel-eligible after Module 6)
- **GATE before starting:** author `docs/api_contracts/flutter_odoo_contract.md` (gap GAP-07): stateless API-key auth (not session cookies), multipart photo upload → `ir.attachment` → triggers AI analysis, concept retrieval, approval POST. No Flutter code before the contract exists.

---

## 6. Definition of Done (every module)

A module is done only when ALL are true:
1. Manifest declares correct dependencies and license `AGPL-3`.
2. All models have `ir.model.access.csv` entries and appropriate record rules.
3. Every public method has a test; every stub has a failing smoke test (Discipline B).
4. Existing tests still pass (Discipline A).
5. The module installs and upgrades cleanly on the disposable local Odoo 19 clone.
6. Any portal/website surface is tested as an anonymous second customer (ACL discipline).
7. No Enterprise dependency; any CE/Enterprise gap is documented in manifest comments.
8. Commits follow Conventional Commits with decision/gap citations.
9. Module-boundary acknowledgment report produced; STOP for "proceed".

---

## 7. Critical Gates Summary (ordered by blocking power)

| Gate | Blocks | Owner action |
|------|--------|--------------|
| **G1** BoM-contents assertion suite passes | Module 2 + all FreeCAD work | Write `test_bom_contents.py` against `shared/southbrook_dims` (final formulas — G2 closed). |
| **G2 — CLOSED 2026-06-09** Peter Tuschak signed off the 7 panel formulas | — (released) | Formulas final. No provisional markers anywhere in the repo. |
| **G2a** Owner explicit go-ahead before Module 2 deploys | Module 2 deployment (NOT build/test) | Owner confirms in writing after the Module 2 acknowledgment report. |
| **G3** Gemini→Odoo JSON contract defined | Module 6, Module 7 | Author `gemini_odoo_contract.md`. |
| **G4** Configuration Engine spec written | Module 7 | Author `config_engine_spec.md`. |
| **G5** FreeCAD 1.0 `freecadcmd` headless confirmed | Module 2 service, templates | Verify AppImage on Docker base OS. |
| **G6** Flutter↔Odoo API contract written | Flutter app | Author `flutter_odoo_contract.md`. |

Never pass a gate by assuming. Verify on disk (Discipline D) and report.

---

## 8. Anti-Patterns — Do NOT

- Do not rebuild or refactor `southbrook_estimating`; extend it via `_inherit` and respect the 7-routine boundary.
- Do not encode parametric panel quantities as static `mrp.bom.line` values — use `create_get_bom`.
- Do not deploy any BoM-producing module before G1 passes.
- Do not model W×H×D as product attributes (they are numeric parametric inputs).
- Do not serve FreeCAD STEP files to the browser for the customer preview — use Three.js geometry from the shared formula source.
- Do not test ACLs as admin; test as an anonymous second customer.
- Do not chain modules without the acknowledgment gate.
- Do not introduce the three known OCA latent defects (public-buyer ACL read, reconfigure IDOR, JSON int-vs-string key).
- Do not assume Gemini returns accurate dimensions — enforce the human-confirmation gate.
- Do not commit secrets; use `.env` and `.env.example`.

---

## 9. Kickoff Instruction

Begin now with **Section 2 (Inspect Before You Build)**. Produce the state inventory and environment confirmation, flag any divergence from this document, then STOP and request acknowledgment before generating Module 0. Do not write module code in your first response — inspect, report, and gate first.
