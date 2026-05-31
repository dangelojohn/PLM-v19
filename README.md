# PLM-v19 — Lightweight PLM for Odoo 19 Community Edition

A thin, **configurator-fit Product Lifecycle Management** addon for Odoo
**19.0 Community Edition**, built for a parametric / configurator-driven
manufacturing shop (kitchen cabinetry) where bills of materials are *generated*
rather than hand-authored from CAD.

> **Why this exists.** Odoo's native PLM (`mrp_plm`) is Enterprise-only; the
> mature free CE option (OmniaSolutions OdooPLM) is a CAD/PDM suite that fights
> a configurator-generated BoM; and the well-fit lightweight ECO modules have no
> v19 build / are proprietary. So this synthesises *only* the capabilities a
> configurator shop actually needs. The full options study is in
> [`docs/PLM-Gap-Fit-Analysis.md`](docs/PLM-Gap-Fit-Analysis.md).

## The module: `addons/southbrook_plm`

| Capability | What it does |
|---|---|
| **Engineering Change Orders** | `southbrook.eco` with a user-configurable Kanban pipeline (`southbrook.eco.stage`), categories (`southbrook.eco.type`), per-stage approval gating, and a full `mail.thread` audit trail. |
| **Template-BoM versioning** | Applying a `bom`-kind ECO copies the target `mrp.bom` to a new `southbrook_version` and archives the prior one. |
| **ECO-governed cut spec** | `southbrook.cut.spec` promotes the parametric geometric constants (panel thicknesses, reveals, rabbet, …) out of code into versioned, approval-gated records. |
| **Engineering documents** | Vendor cut sheets / hardware PDFs / shop drawings attach to the ECO (`ir.attachment`) and ride its approval trail. |
| **Code-change references** | Construction-rule changes stay in code + git; the ECO records the commit SHA / PR via `git_ref`. |

**Deliberately out of scope:** CAD/PDM file vault, CAD bridges, 3D viewers,
part-number generation, routing/operation versioning, per-variant runtime-BoM
versioning, and commercial/pricing change control.

## The cut-spec seam

The host estimating module reads its geometric constants through a seam method,
`_get_cut_constants()`. Standalone it returns code defaults; this addon
**overrides** that seam to return the *active* `southbrook.cut.spec`, with a
fallback to the code defaults when no spec is active. The result: a shop lead can
revise cabinet geometry through an approved change order **with no code deploy**,
and installing the addon changes nothing until a spec is activated.

## Dependency note

`southbrook_plm` `_inherit`s from `southbrook_estimating` (the configurator /
estimating build it was written for). That host module is **not** included in
this public repository, so this addon is published as a **reference
implementation + design rationale**, not a standalone-installable app. The
dependency and the seam contract are documented in
[`addons/southbrook_plm/README.md`](addons/southbrook_plm/README.md).

## Install (in a full environment)

```bash
# requires: odoo 19.0 CE, mrp, and the host southbrook_estimating module
odoo-bin -i southbrook_plm -d <db>
odoo-bin --test-enable --stop-after-init -i southbrook_plm -d <db>   # 14 tests
```

## License

**LGPL-3.0** — see [LICENSE](LICENSE).

## Status

Static-validated (`py_compile` + XML well-formedness + a 14-case test suite).
Not yet installed on a live v19 database at time of publication.
