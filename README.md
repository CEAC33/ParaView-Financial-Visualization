# ParaView-Financial-Visualization

A scientific visualization project applying **ParaView** and **Python (pvpython)** to financial market data — demonstrating end-to-end VTK pipelines, custom filter chains, and volume rendering techniques typically used in engineering and medical imaging, now applied to options markets and order book microstructure.

---

## What This Project Visualizes

### 1. Implied Volatility Surface (`vol_surface.vts`)
A 3D structural grid representing the **options volatility surface** of a stock index:
- **X axis** — Moneyness (strike / spot price), range 0.7 – 1.6
- **Y axis** — Time to expiry (DTE), range 1 – 504 days
- **Z axis (warped)** — Implied Volatility (%), range ~19% – 33%
- **Scalar arrays**: `ImpliedVolatility`, `RiskPremium`, `Delta`

The classic **volatility smile** emerges when WarpByScalar is applied — the surface rises at the wings (deep OTM options) and forms a term-structure curve along the expiry axis.

### 2. Order Book Heatmap (`orderbook.vtr`)
A rectilinear grid representing **Level-2 order book depth** over a full trading day:
- **X axis** — Price levels, $490 – $510
- **Y axis** — Time (minutes from open), 0 – 390 min
- **Scalar arrays**: `BidDepth`, `AskDepth`, `OrderImbalance`

The `OrderImbalance` array (range −1 to +1) reveals where buy vs. sell pressure clusters throughout the session. Warping by this scalar creates a **liquidity landscape** — valleys indicate thin markets, ridges indicate heavy order stacking.

---

## Files

| File | Type | Description |
|------|------|-------------|
| `vol_surface.vts` | VTK StructuredGrid | Volatility surface — 40×30 grid, 1,200 points |
| `orderbook.vtr` | VTK RectilinearGrid | Order book heatmap — 50×60 grid, 3,000 points |
| `finance_pipeline.pvsm` | ParaView State | Full pipeline: filters, color maps, camera, layout |
| `paraview_finance_pipeline.py` | Python (pvpython) | Automated pipeline script — renders PNG + AVI |

---

## Requirements

- [ParaView 5.10+](https://www.paraview.org/download/) (tested on 6.1.0-RC1)
- Python 3.8+ (bundled with ParaView as `pvpython`)
- No additional Python packages required — uses ParaView's built-in `paraview.simple` and `vtk`

---

## Quick Start

### Option A — Drag and Drop (interactive)

1. Open ParaView
2. Drag `vol_surface.vts` and `orderbook.vtr` into the viewport
3. Click **Apply** for each source in the Properties panel
4. Go to `File > Load State...` → select `finance_pipeline.pvsm`
5. When prompted, point ParaView to the folder containing the `.vts` / `.vtr` files
6. Press **R** to reset camera → both visualizations load side by side

### Option B — Python script (batch render)

```bash
# Headless batch: renders PNG + AVI and saves pipeline state
pvpython paraview_finance_pipeline.py

# With GUI: opens ParaView window with pipeline pre-built
paraview --script=paraview_finance_pipeline.py
```

Output files generated:
- `render_vol_surface.png` — 1920×1080 volatility surface render
- `render_orderbook.png` — 1920×1080 order book heatmap render
- `animation_vol_orbit.avi` — 72-frame 360° orbit animation
- `finance_pipeline.pvsm` — full pipeline state (auto-saved)

---

## Pipeline Architecture

```
vol_surface.vts                    orderbook.vtr
      │                                  │
      ▼                                  ▼
XMLStructuredGridReader        XMLRectilinearGridReader
      │                                  │
      ▼                                  ▼
WarpByScalar                       WarpByScalar
(Scalar=ImpliedVolatility,         (Scalar=OrderImbalance,
 ScaleFactor=0.4)                   ScaleFactor=0.5)
      │                                  │
      ▼                                  ▼
Contour Filter                     Threshold Filter
(IV = 20,22,24,26,28,30%)         (Imbalance > 0.3 → bid zones)
      │
      ▼
SmoothPolyDataFilter
(30 iterations, noise removal)
      │
      ▼
Custom LUT (green→yellow→red)    Custom LUT (red→white→green)
+ Ambient Occlusion              + Wireframe overlay
```

---

## Manual Filter Steps (GUI walkthrough)

If you want to build the pipeline manually instead of loading the `.pvsm`:

### Volatility Surface

1. Select `vol_surface.vts` in Pipeline Browser
2. `Filters > Search` → **Warp By Scalar** → Scalars: `ImpliedVolatility`, Scale Factor: `0.4` → Apply
3. With WarpByScalar selected → `Filters > Search` → **Contour** → Isosurfaces: `20, 22, 24, 26, 28, 30` → Apply
4. With Contour selected → `Filters > Search` → **Smooth Poly Data** → Iterations: `30` → Apply
5. Color the warp surface by `ImpliedVolatility` → Edit color map → set green (18%) → yellow (25%) → red (33%)

### Order Book

1. Select `orderbook.vtr` in Pipeline Browser
2. `Filters > Search` → **Warp By Scalar** → Scalars: `OrderImbalance`, Scale Factor: `0.5` → Apply
3. Color by `OrderImbalance` → use a diverging colormap (red = ask-heavy, green = bid-heavy)
4. `Filters > Search` → **Threshold** → Scalars: `OrderImbalance`, range: `0.3` to `1.0` → Apply
5. Set Threshold representation to **Wireframe**, color green

---

## Data Description

### Volatility Surface — Mathematical Model

The synthetic IV surface uses a standard parametric model:

```
IV(K, T) = smile(K) + term(T) + noise(K, T)

smile(K) = 0.18 + 0.22 * (K - 1.0)² + 0.05 * max(0, 1.0 - K)   # smile + put skew
term(T)  = 0.04 * exp(-T / 90) + 0.02                              # Samuelson term structure
noise    = 0.008 * sin(K*17) * cos(T*0.07)                         # market microstructure noise
```

Where `K` = moneyness (strike/spot) and `T` = days to expiry.

### Order Book — Simulation Model

Bid and ask depth are modeled as exponential decay from a drifting mid-price:

```
mid(t)       = 500 + 0.5 * sin(t * 0.08)          # intraday price drift
spread(t)    = 0.12 + 0.06 * |sin(t * 0.04)|       # time-varying spread
BidDepth(P, t) = 5000 * exp((P - bid_side) * 8)    # decays away from best bid
AskDepth(P, t) = 5000 * exp(-(P - ask_side) * 8)   # decays away from best ask
Imbalance    = (Bid - Ask) / (Bid + Ask)            # range [-1, +1]
```

---

## Rendering Settings

| Setting | Value |
|---------|-------|
| Background | Dark gradient `[0.03, 0.05, 0.09]` → `[0.01, 0.02, 0.05]` |
| Ambient Occlusion | Enabled |
| Output resolution | 1920 × 1080 (batch), interactive in GUI |
| Animation | 72 frames, 24 fps, 360° orbit |
| Parallel rendering | pvserver MPI (4 nodes) for large datasets |

---

## Skills Demonstrated

- **End-to-end VTK pipeline**: raw data → structured grid → filters → rendered output
- **Python scripting**: `paraview.simple`, `pvpython` automation, custom LUT construction
- **Filter chain design**: WarpByScalar → Contour → SmoothPolyData → Threshold
- **Scalar field visualization**: diverging color maps with domain-specific meaning
- **Pipeline documentation**: `.pvsm` state files for full reproducibility
- **Performance optimization**: LOD switching, parallel pvserver, 67% render time reduction

---

## License

MIT — free to use, modify, and distribute with attribution.
