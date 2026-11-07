# Battery

Windows Media Player's **Battery** visualization, as implemented in `index.html` of
[visualizations](https://github.com/altrao/visualizations).

## Overview
- The image is an **8-bit palette-index** buffer. Its size is 384×288 by default; the original also offers 256×192
  and 512×384. It is stretched to the window without smoothing. Colour comes only from a 256-entry RGB palette.
- 24 ticks per second. Each tick:
  1. count the timers down, and start a new movement or new effects when they run out;
  2. build 3 more rows of a pending movement table;
  3. run the **PRE** drawing effects;
  4. apply the **movement** (a per-pixel source table);
  5. run the **POST** drawing effects;
  6. **blur + decay**;
  7. step the **palette** morph.
- Every duration below counts ticks.
- `rand()` is an integer 0–32767, `u = rand()/32767`, and `%` is integer modulo. "Truncate" means toward 0.

## Audio
All values are bytes 0–255. Formulas index them as one array `A[]`:

| Range | Content |
|---|---|
| `A[0x000..0x3ff]` | spectrum, left (`F0`), bin 0 = bass |
| `A[0x400..0x7ff]` | spectrum, right (`F1`) |
| `A[0x800..0xbff]` | waveform, left (`W0`), 128 = silence |
| `A[0xc00..0xfff]` | waveform, right (`W1`) |

`bass6 = A[0] + A[2] + A[4] + A[0x401] + A[0x403] + A[0x405]`, used as `bass6/1530`.

## Timers and random choices
- **Movement timer:** when it drops below 1, it is reset to `rand%390 + max(60, H/3)`. A new movement is then picked
  from the pool until it differs from the current one, and gets random parameters. Its table is built 3 rows per tick,
  and when the table is complete the movement takes over through a transition.
- **Effects timer:** when it drops below 1, it is reset to `rand%310 + 90`, and PRE and POST are cleared. Then:
  - `rand%2` times: pick random pool entries until one is an edge drawer, randomise it and append it to PRE;
  - `(rand%4 + 1)/4 + 1` times (2 with probability 1/4, else 1): pick until one is a shape drawer, randomise it and
    append it to PRE if `rand()&1`, else to POST.

  The same effect can be picked twice.
- **Movement pool** (15 slots): Linear, Linear, Thingus, Zoom, RingSpin, Stretch, Tile, Trig, SinShimmer,
  EdgeFalloff, Starburst, Swirl, TrigStretch, Twirlocity, Shiitake. The two Linear slots are separate objects.
- **Drawer pool:**
  - edge drawers: EdgeTrace, EdgeGradiant, CosEdgeGradiant, WaveEdge, SpectrumEdge;
  - shape drawers: CircleWaveform, DotPlane, JDar, Galaxy, JiggyScribble.
- **TileShift suppresses edge drawers in PRE:** a PRE effect is skipped when the movement and the effect are both
  flagged 1. TileShift and the edge drawers are; shape drawers are flagged 2 and other movements 0.

## Presets
- The preset list is **Randomisation**, followed by 25 named presets:
  - brightsphere, cominatcha, dandelionaid, drinkdeep, eletriarnation;
  - cottonstar, gemstonematrix, sepiaswirl, event horizon, illuminator;
  - i see the truth, kaleidovision, green is not your enemy, lotus, relatively calm;
  - sleepyspray, smoke or water?, back to the groove, spider's last moment..., strawberryaid;
  - the world, dance of the freaky circles, my tornado is resting, hizodge, chemicalnova.
- A named preset holds:
  - one movement with its parameters;
  - a PRE list and a POST list of drawers with their parameters;
  - optionally a **locked palette**.

  The table is `BAT_PRESETS` in `index.html`. Parameters a preset doesn't set keep their defaults (0 unless noted).
- **Activating any preset** sets both timers to 0 and rebuilds the blur table. A named preset then:
  - loads its movement and its effects;
  - if its palette is locked, morphs to that palette over 25 ticks and turns the automatic palette changes off.

  Because the timers are 0, the first tick replaces the preset's effects with random ones and starts building a random
  next movement. What lasts is the palette (when locked) and, for the first few seconds, the preset's movement.

## Movements
### Shared rules
- Each movement maps a destination pixel `(x, y)` to a source pixel `(sx, sy)`, truncated to integers.
- `cx = W >> 1` and `cy = H >> 1`.
- The table is `T[y·W + x] = sy·W + sx`. Applying it swaps the buffers and sets `front[i] = back[T[i]]`, nearest pixel.
- **Out of range**, per axis:
  - keep the pixel's own coordinate: Linear, Thingus, Tile, Trig, SinShimmer, Starburst, Swirl, Twirlocity, Shiitake;
  - use 0: RingSpin, Stretch, Zoom, EdgeFalloff, TrigStretch. These pull in row or column 0, the border.
- **Transition from A to B:** 18 intermediate tables. Table k (1–18) uses `src = A + k/19·(B − A)` per axis. Each is
  used for `rand%15 + 1` ticks, then the plain B table takes over.

### Polar movements
`dx = cx − x`, `dy = cy − y`, `r = √(dx² + dy²)`, `a = atan2(dy, dx)`. Each movement computes `r'` and `a'`. **Polar
out** then maps back with a point reflection through the centre:
```
sx = W − cx − trunc(r'·cos a')
sy = H − cy − trunc(r'·sin a')
```
`spin = u·0.1 − 0.05`.

| Movement | Parameters | Map |
|---|---|---|
| RingSpinShift | d1 = spin, d2 = u·0.8 | ring = (H>>1)·d2 (0 → 1e-4); m = r − trunc(r/ring)·ring; r' = r − (m/ring)·m; a' = a + d1 |
| ZoomShift | d1 unused, d2 = u·0.2 − 0.1 | D = √(W² + H²)/2; r' = r − W·d2·(r/D); a' = a |
| StretchShift | d1 = spin, d2 = u·0.3 | k = r/(W>>1); r' = r − k³·trunc(H·d2); a' = a + d1 |
| StarburstShift | d1 = spin, d2 = u·0.3, d3 = an even number 0–40, d4 = rand%2 | k = r/(W>>1), A = trunc(H·d2). d4 = 0: r' = r + (k·sin(trunc(d3)·a))³·A. d4 = 1 (d3 = 0 counts as 1): r' = r ∓ k³·A, minus when trunc(a/(6.28/d3)) is even. a' = a + d1 |
| Shiitake | d1 = u·11 − 1; d2 = π·u·0.05 (1 %: π·u·16); d3 = u·0.05 (1 %: u·8); d4 = 1 + rand%100 (10 %), rand%5 (30 %) or rand%2 (60 %) | r' = r + d1; v = 2π·r'/cy; a' = a + d2·cos(d4·v) + d3·v |
| Twirlocity | d1 = rand%50 + 1, d2 = u·0.6, d3 = rand%10 | h = H/2; k = r/h if d3 = 0, else h/r (0 when undefined); a' = a + d2·cos(k·π·d1); r' = r |
| ThingusShift | d1 unused, d2 = u·0.2 | r' = r + W·d2·((W>>2) − r)/((W>>1) + (W>>2)); a' = a |
| TrigShift | d1 = spin, d2 = u·0.1 − 0.05, d3 = rand%3 | D = √(W² + H²)/2; f = cos a (d3 = 0), sin a (1), or for d3 = 2 cos a on even x and sin a on odd x, using a before spin; f /= 1.57; r' = r − W·d2·(r/D)·f; a' = a + d1 |
| TrigStretchShift | d1–d3 as Trig, d4 = u·0.3 | rT = Trig's r'; rS = r − (r/(W>>1))³·trunc(H·d4); r' = (rS + 2·rT)/3; a' = a + d1 |
| SwirlShift | d1 = spin, d2 = rand%20 − 10, d3 = rand%24 − 12 | x1 = x + trunc(d2·sin(2·d3·3.14/H·y)); y1 = y + trunc(d2·cos(2·d3·3.14/W·x)); polar from (x1, y1): r' = r − d2·sin((a + d1)·d3), then polar out. Then j = rand%4 − 2: sy += (y < cy ? −j : j); sx −= j if x < cx, else sx += j |

### Cartesian movements
| Movement | Parameters | Map |
|---|---|---|
| LinearShift | d1, d2 = rand%6 − 3 | sx = x + trunc(d1), sy = y + trunc(d2) |
| TileShift | d1 = u·0.2 | s = H·d1 (0 → 1e-4); for each of x and y: m = v − trunc(v/s)·s; v' = v − trunc((m/s)³·m) |
| SinShimmerShift | d1 = u·10 − 5, d2 = 2u, d3 = rand%2 | d3 = 0: sy = y − (sin(x·d2) + sin(y·d2))·d1. d3 = 1: sx = x − (sin(y·d2) + sin(x·d2))·d1 |
| EdgeFalloffShift | d1 = u·0.1, d2 = rand%4 | 0: sx = (1 + d1)·x; 1: sy = (1 + d1)·y; 2: sx = x − trunc(d1·(W − 1 − x)); 3: the same on y with H |

## Drawing effects
All effects write palette indices. "Border lines" are the four edges of the image, drawn with one value.

### Edge drawers
| Effect | Parameters | Draws |
|---|---|---|
| WaveEdge | d1 = rand%2 | Source: spectrum if d1, else waveform; left channel on the top and left edges, right channel on the bottom and right edges. Row 0 x = src0[x % 1024] for x < W−1; row H−1 = src1[x % 1024]; column 0 = src0[y & 1023]; column W−1 = src1[y & 1023] for y < H−1 |
| EdgeGradiant | none; v = 0, step = +1 | If v (unsigned) > 255, the step reverses; v += step; border lines with the low byte of v |
| CosEdgeGradiant | d1 = u·0.09 | phase += d1 (back to 0 past 2π); border lines with the low byte of trunc(253·cos(phase) + 1) |
| SpectrumEdge | d1 = rand%10, d2 = 2·(rand%10) + 2, d3 = rand%11 + 4 | A ring of d3 sums: slot = (slot + 1) % d3, sum[slot] = Σ over j = 0, 2, 4 … < d2 of (A[j] + A[j + 0x401]). v = (Σ sums)/(d3·d2) as a byte; v = ~v if d1 = 0, v = trunc(0.9·v) if d1 > 5. Border lines with v |
| EdgeTrace | d1 = rand%30 + 35 | o = (o + 1) % d1 + 3. Clockwise around the border from offset o, every d1 px: a 4-px dash (0xf5, 0xff, 0xff, 0xf5) on the edge plus 2 px of 0xf5 one pixel inward |

### Shape drawers
**CircleWaveform**
- Parameters:
  - `d1 = rand%4`, the colour mode;
  - `d2` = 1–3 copies, usually 2;
  - `d3 = u·0.6 + 0.05`, the share of the waveform drawn;
  - base radius `R0 = 20`, `phase = 0`.
- Draw:
```
S = trunc(A[0]/256 · (H>>1))
for each copy c < d2:
    centre = screen centre, or for d2 > 1: centre + (W>>3)·(cos, sin)(2π/d2·c + phase)
    n = trunc(1024·min(d3, 1))
    for i in 1..n−1: θ = i/n·π
        upper = centre + (S·W0[i]/256 + R0)·(cos θ,  sin θ)
        lower = centre + (S·W1[i]/256 + R0)·(cos θ, −sin θ)
        line from the previous upper point and from the previous lower point
        value: d1 0: |w − 128|·2   1: w   2: 255   3: v = i/n·768, inverted when bit 8 is set (all as bytes)
phase += 0.049087
```

**JiggyScribble**
- Parameters:
  - `d1 = rand%100 + 4`, the epicycle radius;
  - `d2 = rand%200 + 40`;
  - `d3 = rand%1000 + 300` points;
  - `d4 = rand%20 + 1`, the epicycle frequency;
  - `d5 = u·0.6 + 0.05`, the spin;
  - `d7 = rand%10`.
- Draw:
```
phase += d5 (wrapped at 2π);  R = bass6/1530 · (d2 − d1)
for t = 0, 2π/d3, …, < 2π:
    p = R·(cos t, sin t) + d1·(cos d4·t, sin d4·t), rotated by phase, + centre
    d7 = 9: line from the previous point, value 255;  otherwise a dot, value 255
```

**JDar**
- Parameters:
  - `d2 = rand%4`, the sign mode;
  - `d3 = rand%2` (0 also draws mirrored);
  - `d4 = 2^(rand%4 + 2)` segments;
  - `d5 = u·1.5 + 0.3`;
  - `d6 = rand%2` (0 = connected);
  - colour `c` = a random byte;
  - an **anchor** at `(rand%200, rand%200)` with velocity `(u, u)`, bouncing in the box `[0, 472]×[0, 312]`. The box
    doesn't depend on the image size.
- Each tick:
```
anchor += vel·6, bouncing
s = (A[0] + A[1] + A[0x400])/9000;  phase += s (wrapped at 2π);  c = (c + trunc(6s)) mod 256
E = Σ k<512 of (A[2k] + A[2k + 0x401]);  step = 1024/d4;  ρ = rand % (W>>1);  C = 0;  P = centre
for segment i < d4:
    b = Σ j<step/2 of (A[0x400 − i·step − 2j] + A[0x7ff − i·step − 2j]);  C += trunc(b)
    δ = b/(step·255)·d5, negated when:
        d2 = 0: i is even;  d2 = 1: a toggle flipped each segment is 0;  otherwise: rand%2 = 1
    Q = centre + (C·ρ/E)·(cos, sin)(phase + δ)
    arc(P → Q around the anchor, n = 100, value c → 255, connected = (d6 = 0), shortest way)
    if d3 = 0: the same arc point-mirrored (W − x, H − y, anchor mirrored)
    P = Q
```

**Galaxy**
- Parameters:
  - `d1 = rand%5 + 1` arms;
  - `d3 = (u·0.4 + 0.05)/d1`;
  - `d4 = rand%56 + 200` points per arc;
  - `d5 = rand%10` (0 = connected);
  - `d6 = rand%150`;
  - `d7 = u·6e-7`;
  - `d8 = u·0.3`;
  - offset `(ox, oy) = (rand%200, rand%200)` if `d6 <= 3`, else 0;
  - velocity `(u·8192, u·8192)`.
- Each tick:
```
g = bass6/1530
if d6 < 2 (moving centre):
    n̂ = (vy, −vx)/|v|;  q = ox² + oy²
    vx += n̂x·g − ox·q·d7 − vx·d8;   vy += n̂y·g − oy·q·d7 − vy·d8
    (ox, oy) += vel·0.6, bouncing inside [−W, W − 40]×[−H, H − 40]
phase += g·d3 (wrapped at 2π)
for arm i < min(d1, 50):
    R  = (A[i] + A[i+2] + A[i+4] + A[i+0x401] + A[i+0x403] + A[i+0x405])/1530 · (H + d6)
    R2 = R·A[i + 0xc00]/255;   k = A[i + 0x800]/255;   angle = i·π/n + phase
    C = centre + (ox, oy);  P1 = C + R·(cos, sin)(angle)
    P2 = C + (R2·cos + R·sin·(1 − k), R2·sin − R·cos·(1 − k))
    arc(C → P1 around P2, n = d4, value 255 → d4, connected = (d5 = 0)), and the same point-mirrored through C
```

**DotPlane**: a scrolling 3-D spectrum landscape
- Parameters:
  - `d1 = rand%28 + 20`, the grid size N;
  - `d2` = rotation axes as bits, each set with probability 1/4: 1 = X, 2 = Y, 4 = Z;
  - camera distance `0.6·W`.
- **Grid:** N rows × N columns of points `(x, h, z)`, each with a fall speed `v`.
- **Each tick, a new row** replaces the oldest:
```
bins: b0 = 1, b(j+1) = max(b(j) + 1, trunc(b(j)·1.096))
column j: x = j − N/2,  z = −N/2,  h = (max(F0[b(j)], F1[b(j)]) >> 2)²·0.015,  v = 0
```
- **The other rows:** `z += 1`; `if h > 0: h −= v, clamped at 0; v += 0.01`.
- **Rotation:** each enabled axis counts 0–299 and turns by `θ = 2π·k/300`.
  `Model = Rx·Ry·Rz·T` (row vectors), where T moves the grid `0.6·W` along the unit vector from the target to the eye.
- **Camera:**
  - eye at `(W/2, H/2, H/2)`, looking at the origin, up = +y;
  - vertical field of view 60°, far plane 1000;
  - square viewport `min(W, H)`, centred horizontally.
- **Plot:** each point is projected, and points in front of the camera inside the image are written with value 0xfe.

### Arc primitive
`arc(P0 → P1 around C, n, v0 → v1, connected, mode)`:
```
r0 = |P0 − C|, r1 = |P1 − C|;  a0, a1 = their angles in [0, 2π);  da = a1 − a0
mode 1: da = −(2π − da)    mode 2: keep    mode 4: go the other way when da < π
other modes (0, 3): go the other way when da >= π (the shortest way)
for i in 0..n−1:
    r = r0 + (r1 − r0)·sin(i·π/(2n));  angle = a0 + i·da/n;  value = v0 + i·(v1 − v0)/n
    plot C + r·(cos, sin)(angle), or, when connected, draw a line from the previous point (the first one is P0)
```
Lines are clamped into the image and drawn with one value.

## Blur + decay
```
LUT[s] = max(1, floor(s/5) − 1)    for s = 0..1279     (never 0, at most 254)
new[i] = LUT[c + up + down + left + right]              (read from the old buffer, written to the other one)
```
- Neighbours are linear indices: left = `i − 1`, right = `i + 1`, up/down = `i ∓ W`.
- So the side edges wrap into the neighbouring row, and the top row reads the bottom row and vice versa.
- Every tick brightness spreads out and drops by about one step, so trails fade into index 1, the background.

## Palette
- Entries are R, G, B. There are three tables: START, DISPLAY and TARGET.
- **A new palette:** START = DISPLAY, TARGET = new, duration D, remaining R = D.
- **Each tick while R > 0:** `t = (D − R)/D` (1 if D <= 0); `DISPLAY = START + trunc((TARGET − START)·t)` per channel;
  `R−−`. When R reaches 0 (and the palette isn't locked), wait `rand%600` ticks, then generate a new palette.
- **Generator:**
  - `n = 2 + (rand%10 ≠ 0 ? rand%4 : rand%10)` keys;
  - key 0 is dark, `(rand%32, rand%32, rand%32)`;
  - key 1 is bright, `(255 − rand%32, …)`;
  - the other keys are random 0–255 per channel;
  - keys are sorted by R + G + B, ascending, or descending 1 time in 10;
  - `D = rand%250`.
- **Gradient:** `span = floor(256/(n − 1))`; segment i covers `a = i·span` to `b = (i + 1)·span` (b = 255 for the
  last); `TARGET[idx] = lerp(key i, key i+1, (idx − a)/(b − a + 1))`.

## Playback states
- **Playing:** a normal tick.
- **Paused:** the image is shown without ticking.
- **Stopped:** for 300 ticks, blur + decay with border lines of value 1. After that the screen is filled with
  palette[1].

## Implementation notes
- `makeBattery(W, H)` in `index.html` is the simulation, with no DOM. `drawBattery(dt)` runs it at 24 ticks/s and
  stretches the image to the window with smoothing off.
- One mono analyser feeds both channels. The browser's spectrum bytes are dB-scaled, not calibrated.
- The page always counts as playing, so the paused and stopped screens aren't shown.
