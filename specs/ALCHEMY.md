# Alchemy

Windows Media Player's **Alchemy** visualization, as implemented in `index.html` of
[visualizations](https://github.com/altrao/visualizations).

## Overview
- One persistent 32-bit RGB image is the whole state. Nothing clears or fades it.
- Every tick (14 per second):
  1. the image is copied through a per-pixel **warp** table and blurred on the way;
  2. up to two **drawing effects** draw on top;
  3. **Hover** may zoom the displayed picture. The zoom is not fed back.
- Every duration below counts ticks.
- Every size and speed is in pixels. The image is window-sized, so the window size changes the look.
- `rand(a, b)` is a uniform float, `randInt(a, b)` a uniform integer (inclusive), and `lerp(c, old, t) = c + (old − c)·t`,
  so `t` is the share of the old value kept.

## Audio
Input per tick, all bytes 0–255:
- the spectrum `f0[1024]` (left) and `f1[1024]` (right), bin 0 = lowest;
- the waveform `w0[1024]` and `w1[1024]`, where 128 = silence.

```
bass   = (f0[1] + f0[3] + f0[5] + f1[2] + f1[4] + f1[6]) / 1200
treble = (f0[501] + f0[503] + f0[505] + f1[502] + f1[504] + f1[506]) / 600
hist[i++ % 30] = bass;  avg = mean(hist)
sinceBeat++; sinceDrop++
beat     = sinceBeat >= 10 && bass > 1.1·avg                              (resets sinceBeat)
bassdrop = beat && bass > 0.9 && bass > 1.2·avg && sinceDrop > 20          (resets sinceDrop)
```

## Colours
- A **colour slot** eases from colour `a` to colour `b` over T ticks: `c = lerp(a, b, sin(f/T·π/2))`.
  On arrival `a = b`, a new random `b` is picked, and `T = randInt(1, 100)`.
- A random colour is uniform 24-bit RGB. There is no palette.
- Each drawing effect owns two colour slots. Both advance once per tick.
- **Line gradient:** along a line the colour bounces k times between the effect's two current colours. The colour is
  constant within each sample segment.

## Pen: every line is an oscilloscope trace
`line(A, B)`:
```
n = min(samples, max(|dx|, |dy|) + 1)            // samples = the effect's point count
perp = unit vector at angle(B − A) + π/2
for i in 0..n−1:
    t = i/n;  P = A + (B − A)·t + perp·level(i)
    draw a clipped Bresenham segment from the previous P to this P
    if mirrored: draw the same segment for P' = A + (B − A)·t − perp·level(i)
```
`level(i)`:
```
j = i;  h = n/2;  if j > h: j = 2h − j          // the trace is symmetric about its middle
w = (w0[j] + w1[j]) / 2
v = amp · (w − 128)/128                         // amp in pixels
envelope 0: v·|sin(i/n · loops · π)|   1: v·(triangle peaking at n/2)   2: v
```
`lineThrough(C, angle, len)` = `line(C + dir·len/2, C − dir·len/2)`.

**Plotting a pixel** (only for 1 ≤ x ≤ W−1 and 1 ≤ y ≤ H−1):

| Plot mode | Effect |
|---|---|
| 0 | `px = lerp(c, px, 0.9)`: only 10 % colour per hit, so lines build up over ticks |
| 1 | `px = c`; each of the 4 neighbours becomes `lerp(c, nb, 0.3)` |
| 2, 3 | `px = c`; each neighbour becomes `lerp(c, nb, α)` (Ocean α = 0.1, Scale α = 0.5) |

## Scheduler
There are eight slots `(category, min, max, durMin, durMax, flags)`, each with its own countdown. When a countdown runs
out:
```
clear the slot's active effects
count    = min of (flags&2 ? 10 : flags&1 ? 3 : 1) draws of randInt(min, max)
duration = randInt(durMin, durMax)                // the new countdown
up to 100 tries while count > 0:
    pick a random effect of the slot's category that no other slot is using; accept it if rand(0, 1) <= weight
    the first activation initialises it, later ones re-randomise its parameters
    ramp in over rampLen = min(randInt(10, 90), duration/3) ticks; ramp out once the countdown <= rampLen
```

| Effect | Category | Weight |
|---|---|---|
| Warp | 3 | 1.0 |
| Ocean | 4 | 1.0 |
| Scale | 4 | 0.5 |
| Balls | 4 | 0.9 |
| Hover | 7 | 0.5 |

| Slot | Result |
|---|---|
| `(3, 1, 1, 90, 500, 0)` | a warp change every 90–500 ticks |
| `(4, 1, 2, 50, 350, 2)` | one drawing effect (the minimum of 10 draws is almost never 2) for 50–350 ticks |
| `(4, 1, 3, 50, 350, 2)` | a second drawing effect, different from the first |
| `(7, 0, 1, 50, 400, 1)` | Hover; count is the minimum of 3 draws of {0, 1}, so it is active 1/8 of the time |

The other four slots ask for categories 5 and 6, which have no effects, so they do nothing.

## Warp
### The copy
The warp keeps a table `map[dst] = src index` (nearest pixel). Each tick `img = blur(prev[map])`, written into a
separate buffer that becomes the next tick's source. The blur treats the image as one linear buffer: the neighbours of
`i` are `i ± 1` and `i ± W`, wrapping at row and image ends. Per channel:
```
q   = floor((l + r + u + d) / 4)
out = floor((3q + c) / 4)
```
Repeating this blur every tick gives the soft colour washes, and repeating the warp gives the trails.

### Choosing the next warp
- There are 8 warps: two each of Shift/stretch, Swirl, Blocks and Reflect.
- The next warp is a random unused one with fresh parameters.
- One time in 5 it is **combined** with a second warp: `src = (p + w2(w1(p))) >> 1`. Reflect never combines.
- **Out of range:** if a warp's `src.x` (or `src.y`) leaves the image, Swirl and Blocks keep the destination coordinate,
  and Shift/stretch and Reflect use 0.

### Transition
The old table eases into the new one in **22 steps**: step k uses `old + (new − old)·(k + 1)/23` per axis.
- **Mode** is the minimum of 3 draws in 0–2:
  - 0: `k = floor(sin(f/(22·T)·π/2)·22)`, a sine ease over 22·T ticks;
  - 1: k increases by 1 every T ticks;
  - 2: k increases by 1 with probability 1/T per tick.
- **T** is `randInt(10, 14)` (2/3 of the time) or `randInt(1, 15)` (1/3). It is forced to 1 when a Reflect warp is
  involved, so reflections snap in.
- A typical transition lasts about 260 ticks, so the warp is nearly always mid-transition.

### Warp 1: Shift / stretch
Parameters:
- `dx, dy = randInt(−3, 3)`, not both 0;
- stretch 2/3 of the time;
- `s1, s2 = rand(0, 0.1)`;
- `edge = randInt(0, 7)`;
- `wob = randInt(0, 2)`;
- `k = randInt(1, 15)`.
```
stretch:
  edge 0: x' = (1 + s1)·x               1: y' = (1 + s2)·y
       2: x' = x − s1·(W − 1 − x)       3: y' = y − s2·(H − 1 − y)
       4: 0 and 1     5: 2 and 3     6: x as 0, y as 3     7: x as 2, y as 1
shift:  x' = x + dx,  y' = y + dy
wob 1: x' += round(sin(y'/H·k·π)·3·dy)       wob 2: y' += round(sin(x'/W·k·π)·3·dx)
```

### Warp 2: Swirl
Parameters:
- sine mode 50 %;
- `k = randInt(1, 15)`;
- `rot = ±0.22` in sine mode, else `±0.1`;
- `K = int(H·rand(0.05, 0.3))`;
- the centre is random, `(rand(0.05, 0.95)·W, rand(0.05, 0.95)·H)`, half the time, else the screen centre;
- `maxR` = the distance from the centre to the farthest corner.
```
d = p − c;  r = |d|;  q = r/maxR
θ' = atan2(d) + (sine ? rot·sin(k·π·q) : rot·q)
r' = r − K·q³
src = c + r'·(cos θ', sin θ')
```

### Warp 3: Blocks
Parameters: `T = randInt(1, 40)`, `s = randInt(1, 4)`, vertical-only 1/4 of the time. With `P = 2T`, `bx = x % P`,
`by = y % P`:
```
vertical only:  y' = (bx <= T) ? y − s : y + s
else if by > T: if (W − x >= T || bx <= by − T) x' = x + s  else y' = y + s
else:           if (x < T && bx > by)          y' = y + s  else x' = x − s
```

### Warp 4: Reflect + sub-warp
Parameters:
- `mode = randInt(0, 3)`;
- the sub-warp is a fresh Swirl or Shift/stretch (50 % each);
- `R = randInt(5, 254)`;
- `m = randInt(1, 25)`;
- `cell = (W/2 − R)/m`;
- the centre `c = (W/2, H/2)`.
```
mode 0, 4-way mirror:  f = 0; if x > cx { x = W − x; f = 1 }; if y > cy { y = H − y; f = 1 }; if !f: sub-warp
mode 1, diagonals:
  if y·W >= x·H:
     if y·W >= H·(W − x): y = H − y; if x > cx: x = 2cx − x
     else:                x = cx − x;  y = (y >= cy) ? y − cy : cy − y
  else:
     if y·W < H·(W − x):  if x < cx: sub-warp, else x = 2cx − x
     else:                x = W + cx − x;  y = (y >= cy) ? y − cy : cy − y
mode 2, tiles:  if |p − c| > R: x = (cx − R) + round((x % (cell + 2))·2R/cell), y likewise with cy
                else: sub-warp
mode 3, circle mirror:  r = |p − c|; if r >= D/4 (D = diagonal): r' = D/2 − r at the same angle
                        else: as mode 0
```
Mirroring is part of the feedback table, not done at display time.

## Drawing effects
### Ocean: a rotating scope line through the centre
Parameters:
- `plotMode = randInt(0, 2)`;
- `samples = randInt(10, 521)`;
- `envelope` = the minimum of 3 draws in 0–2;
- `loops = randInt(1, 4)`;
- `amp = H·rand(0.1, 0.425)`;
- `spin = rand(−0.13, 0.13)`;
- `spinMode = randInt(0, 3)`;
- bass flex 50 %;
- mirrored 1/3;
- cross line 1/3, with `crossMode = randInt(0, 2)`;
- `angle2 = rand(−π/2, π/2)`.
```
len = diagonal · (bassFlex ? bass : 1)
pen: 2·loops colour bounces, plotMode, α 0.1, samples, amp, envelope, loops, mirrored
lineThrough(centre, angle + π/2, len)
if cross:
   crossMode 0: angle2 −= spin·bass·0.5; a = angle2 + angle      (counter-rotates)
   crossMode 1: a = angle2 + angle                                (fixed offset)
   crossMode 2: a = angle                                         (perpendicular to the main line)
   lineThrough(centre, a, len)
spinMode 0: angle += bass²·spin   1: angle += spin·bass   2: angle += spin   3: no spin
```

### Scale: a spinning star polygon
Parameters: `size = rand(0.2, 1)`, `spinAmt = rand(−0.2, 0.2)`, `n = randInt(3, 12)`.
```
R = bass·(H/2)·size
θ += treble·spinAmt  (wrapped at 2π)
step = 2π/n·(floor(n/2) + 1)          // even n gives a smaller polygon
pen: 3 colour bounces, plot mode 3, α 0.5, 50 samples, amp 50 px, sine envelope, 2 loops
P0 = centre + R·(cos θ, sin θ); for i in 0..n: P1 = centre + R·(cos(θ + (i+1)·step), sin(θ + (i+1)·step)); line(P0, P1); P0 = P1
```

### Balls: two point-mirrored balls
Parameters:
- random position, velocity `rand(−2, 2)` per axis;
- `R1 = rand(9, 30)`;
- `damp = rand(0.5, 0.9)`;
- the colour slots start with 5–54-tick eases.

Each tick:
```
pos += vel; per axis: outside [0, W] or [0, H] → vel = −vel (bounced), else vel *= 0.8
if not bounced: vel.x += ±bass, vel.y += ±bass       (random signs)
vr += R1 − r;  vr = (vr + 0.8·bass)·damp;  r += vr;  rd = int(r) + 1        // springy radius
A = round(pos);  B = (W − A.x, H − A.y)
wgt = bass;  rA = rB = rd
if beat:            wgt = 0.4; invert = !invert; rA = rB = 2·rd; lineTicks = 5
else if lineTicks:  rA = round(lineTicks·0.4·rd)
if lineTicks: pen (randInt(1, 6) bounces, plot mode 1, 500 samples, amp 50, sine envelope, 2 loops); line(A, B); lineTicks−−
disc(A, rA, wgt); disc(B, rB, wgt)
```
`disc(C, r, wgt)`: for every pixel in the square `C ± r`, with `v = (r² − d²)/r`:
- if `v >= 1`: `px = lerp(col1, px, cos((v/r)²·π/2)·wgt)`. The centre is solid `col1` (or `0xFFFFFF − col1` when
  inverted), and the old image shows through toward the edge in proportion to bass.
- if `|v| < π`: `px = lerp(px, col2, (cos v + 1)/2)`, a rim of `col2` about 3 px wide.

## Hover: display zoom
Hover only changes the source rectangle of the final copy to the screen. Parameters: `jump = rand(0.13, 0.25)`,
`h = rand(0.05, 0.2)`, `period = randInt(25, 55)`; `s` starts at 1.
```
if bassdrop and not ramping out: s = jump; f = 0                // a 4–8× zoom punch
target = sin(2π·(f % period)/period)·h + 1 − h
ramping out: u = 1 − progress/rampLen; target = target·u + 1 − u; a = 15u      otherwise a = 15
s = min(1, (target·(16 − a) + a·s)/16);  f++
source rectangle = centred, s·W × s·H, stretched to the whole output
```
Normally the zoom swings between 1× and 1/(1 − 2h).

## Implementation notes
- `makeAlchemy(W, H)` in `index.html` is the simulation, with no DOM. `drawAlchemy(dt)` runs it at 14 ticks/s and
  draws Hover's rectangle stretched to the window, smoothed.
- The image is window-sized and capped at 1280 px wide for CPU cost.
- One mono analyser feeds both channels. The browser's spectrum bytes are dB-scaled, so the bass thresholds aren't
  calibrated.
- Key `n` moves to a new warp.
