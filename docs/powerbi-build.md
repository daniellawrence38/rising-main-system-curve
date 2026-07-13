# Power BI build reference

Everything needed to rebuild the calculator as a Power BI dashboard: what-if parameters, the
flow-axis table, and the full DAX measure list. The `.pbix`/`.pbip` isn't generated here — build
it in Power BI Desktop following this, and save the `.pbip` source into `../powerbi/`.

## 1. What-if parameters
`Modeling → New parameter → Numeric range`. Each creates a one-row table plus a `[<Name> Value]`
measure — **rename that measure** to the clean name below (the DAX references these).

| Measure | Min | Max | Incr | Default |
|---------|-----|-----|------|---------|
| `D mm` | 50 | 1200 | 5 | 150 |
| `L m` | 10 | 5000 | 10 | 500 |
| `k mm` | 0.05 | 3 | 0.05 | 0.6 |
| `SumK` | 0 | 20 | 0.5 | 2 |
| `Discharge` | 0 | 100 | 0.1 | 25 |
| `Overflow` | 0 | 100 | 0.1 | 12.5 |
| `PumpWL` | 0 | 100 | 0.1 | 10 |
| `Qmin` | 0 | 500 | 1 | 15 |
| `Qmax` | 0 | 500 | 1 | 30 |
| `nu6` (ν ×10⁻⁶) | 0.8 | 1.6 | 0.01 | 1.31 |
| `H0` | 0 | 150 | 0.5 | 42 |
| `Q1` | 0 | 500 | 1 | 25 |
| `H1` | 0 | 150 | 0.5 | 33 |
| `Q2` | 0 | 500 | 1 | 50 |
| `H2` | 0 | 150 | 0.5 | 18 |

## 2. Flow-axis table (disconnected)
`Modeling → New table`. Keep it **unrelated** to everything else. Rename the column to `Q_ls`.
Use a 1 L/s step so the pump line and the intersection resolve sharply.

```DAX
FlowAxis = GENERATESERIES(0, 200, 1)   -- Q in L/s
```

## 3. Core measures

```DAX
Area m2 = PI() / 4 * ( [D mm] / 1000 ) ^ 2

Hs max = [Discharge] - [PumpWL]     -- well drawn down, biggest lift
Hs min = [Discharge] - [Overflow]   -- well at overflow, least lift

Vel min = DIVIDE( [Qmin] / 1000, [Area m2] )
Vel max = DIVIDE( [Qmax] / 1000, [Area m2] )
```

## 4. System curve (one measure per static case)

```DAX
System head (max static) =
VAR Dp  = [D mm] / 1000
VAR A   = PI() / 4 * Dp ^ 2
VAR kp  = [k mm] / 1000
VAR nu  = [nu6] * 1E-6
VAR Q   = DIVIDE( SELECTEDVALUE( FlowAxis[Q_ls] ), 1000 )
VAR v   = DIVIDE( Q, A )
VAR Re  = DIVIDE( v * Dp, nu )
VAR lam = DIVIDE( 0.25, POWER( LOG( kp/(3.7*Dp) + DIVIDE(5.74, POWER(Re,0.9)), 10 ), 2 ) )
VAR hf  = ( lam * DIVIDE([L m], Dp) + [SumK] ) * v^2 / (2 * 9.81)
RETURN [Hs max] + IF( v > 0, hf, 0 )
```

Duplicate as `System head (min static)`, swapping the last line to `[Hs min] + IF( v > 0, hf, 0 )`.

Dynamic-head and total-head KPIs (same friction body, `Q = [Qmax]/1000` or `[Qmin]/1000`,
returning `hf` only):

```DAX
Dyn head @Qmax = ...   -- friction body with Q = [Qmax]/1000, RETURN IF(v>0, hf, 0)
Dyn head @Qmin = ...   -- friction body with Q = [Qmin]/1000, RETURN IF(v>0, hf, 0)
Total head max = [Hs max] + [Dyn head @Qmax]
Total head min = [Hs min] + [Dyn head @Qmin]
```

## 5. Pump curve (quadratic H = a0 + a1·Q + a2·Q²)

```DAX
Pump a0 = [H0]
Pump a2 = DIVIDE( [Q1]*([H2]-[H0]) - [Q2]*([H1]-[H0]), [Q1]*[Q2]*([Q2]-[Q1]) )
Pump a1 = DIVIDE( ([H1]-[H0])*[Q2]*[Q2] - ([H2]-[H0])*[Q1]*[Q1], [Q1]*[Q2]*([Q2]-[Q1]) )

Pump head =
VAR Q = SELECTEDVALUE( FlowAxis[Q_ls] )
VAR h = [H0] + [Pump a1]*Q + [Pump a2]*Q*Q
RETURN IF( h >= 0, h )
```

## 6. Velocity window (0.75 & 2.5 m/s as flows)

```DAX
Q @0.75 = 0.75 * [Area m2] * 1000
Q @2.5  = 2.5  * [Area m2] * 1000
```

Show it as two **X-axis constant lines** (Analytics pane; set each value via the `fx` button to
the measures above, shade between them), or as a faint area series that returns the y-max when
`Q` is inside the window and `BLANK()` otherwise.

## 7. Duty point (pump ∩ system)

DAX can't root-find, so scan `FlowAxis` for the flow where pump and system heads are closest.

```DAX
Duty flow (max static) =
VAR Dp = [D mm]/1000
VAR A  = PI()/4 * Dp^2
VAR kp = [k mm]/1000
VAR nu = [nu6]*1E-6
VAR t =
    ADDCOLUMNS( FlowAxis,
        "@diff",
            VAR Q   = FlowAxis[Q_ls]/1000
            VAR v   = DIVIDE( Q, A )
            VAR Re  = DIVIDE( v*Dp, nu )
            VAR lam = DIVIDE( 0.25, POWER( LOG( kp/(3.7*Dp) + DIVIDE(5.74,POWER(Re,0.9)),10),2) )
            VAR hf  = ( lam*DIVIDE([L m],Dp) + [SumK] ) * v^2 / (2*9.81)
            VAR sysH  = [Hs max] + IF( v>0, hf, 0 )
            VAR pumpH = [H0] + [Pump a1]*FlowAxis[Q_ls] + [Pump a2]*FlowAxis[Q_ls]^2
            RETURN ABS( pumpH - sysH )
    )
RETURN MINX( TOPN( 1, t, [@diff], ASC ), FlowAxis[Q_ls] )
```

Duplicate as `Duty flow (min static)` — swap `[Hs max]` → `[Hs min]` (gives the higher-flow crossing).

```DAX
Duty vel (max static) = DIVIDE( [Duty flow (max static)]/1000, [Area m2] )
Duty vel (min static) = DIVIDE( [Duty flow (min static)]/1000, [Area m2] )

Duty velocity OK =
VAR lo = MIN( [Duty vel (max static)], [Duty vel (min static)] )
VAR hi = MAX( [Duty vel (max static)], [Duty vel (min static)] )
RETURN SWITCH( TRUE(),
    lo < 0.75, "Below self-cleansing",
    hi > 2.5,  "Above 2.5 m/s",
    "Within 0.75-2.5 m/s" )
```

## 8. Mark the duty point on the chart

Markers-only series that is non-blank only at the duty flow:

```DAX
Duty marker (max static) =
IF( SELECTEDVALUE( FlowAxis[Q_ls] ) = ROUND( [Duty flow (max static)], 0 ),
    [System head (max static)] )
```

Duplicate for min static. On those series: markers on, line off, distinct colour.

## 9. Visuals
- **Line chart** — X: `FlowAxis[Q_ls]`; Y: `System head (max static)`, `System head (min static)`,
  `Pump head`, `Duty marker (max/min static)`.
- **Cards** — static/dynamic/total head, duty flow, duty velocity; conditional-format the duty
  velocity card on `[Duty velocity OK]`.
- **Sliders** — all what-if parameters.

> Accuracy: the DAX intersection resolves to the `FlowAxis` step (±0.5 L/s at 1 L/s). The HTML
> calculator interpolates, so treat it as the reference if the two differ by a fraction of a L/s.
