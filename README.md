# Rising Main System Curve

Hydraulic tooling for pumping-station rising mains. Given the pipe, levels, flow range and
pump curve, it computes the **system curve**, overlays the **pump curve**, finds the **duty
point(s)**, and checks flow velocity against the **0.75–2.5 m/s** self-cleansing window.

Two deliverables live here:

| What | Where | How it's published |
|------|-------|--------------------|
| Interactive calculator | [`index.html`](index.html) | GitHub Pages (static site) |
| Power BI dashboard | build reference in [`docs/powerbi-build.md`](docs/powerbi-build.md); `.pbip` source in [`powerbi/`](powerbi/) | Power BI Service (`Publish` from Desktop) |

## The calculator (`index.html`)

A single self-contained HTML file — no build step, no dependencies. Open it locally or serve
it via GitHub Pages. Inputs recalculate live.

**Hydraulic model**
- Area `A = π/4·D²`, velocity `v = Q/A`
- Static head: `max = Discharge − Pump water level`, `min = Discharge − Overflow level`
- Friction (Darcy–Weisbach) `hf = (λ·L/D + ΣK)·v²/2g`, with `λ` from the **Swamee–Jain**
  explicit form of Colebrook–White
- Pump curve fitted as a quadratic through shut-off head + two duty points
- Duty point = pump ∩ system, solved for both static extremes (a duty-flow *range*)

### Publish to GitHub Pages
1. Push this repo to GitHub.
2. **Settings → Pages → Build and deployment → Deploy from a branch**, select your default
   branch and `/ (root)`.
3. Live at `https://<user>.github.io/<repo>/` within a minute.

## The Power BI dashboard

GitHub is **source control only** — the dashboard itself is published to the Power BI Service.
Save the report from Power BI Desktop in **`.pbip` (Power BI Project)** format into
[`powerbi/`](powerbi/) so it version-controls as readable text. See
[`docs/powerbi-build.md`](docs/powerbi-build.md) for every parameter, table, and DAX measure.

## Engineering conventions
- Colebrook–White / Darcy–Weisbach friction (UK/AU water-industry standard for rising mains)
- Self-cleansing velocity **0.75 m/s**, upper design velocity **2.5 m/s**
- Levels in m AOD; flows in L/s; kinematic viscosity default 1.31×10⁻⁶ m²/s (≈10 °C)
