# Wormhole NOC Topology Visualizer

An interactive 3D visualization tool for understanding the Network-on-Chip (NOC) architecture in Tenstorrent Wormhole processors.

## Overview

Wormhole chips contain a 10×12 grid of nodes connected by two independent, **unidirectional** networks:

| NOC | Direction | Color |
|-----|-----------|-------|
| **NOC0** | +X (right), +Y (down) | Cyan |
| **NOC1** | -X (left), -Y (up) | Orange |

The grid wraps around as a **torus** - packets can travel off one edge and appear on the opposite side.

## Node Types

| Type | Color | Description |
|------|-------|-------------|
| **DRAM** | Green-Blue gradient | Memory banks (D1-D6) at x=0 and x=5 |
| **Tensix** | Purple | Compute cores |
| **Ethernet** | Indigo | External connectivity (rows 0 and 6) |
| **PCIe/ARC** | Gray | Host interface and control |

### DRAM Bank Layout

```
x=0 (Left Column)        x=5 (Right Column)
┌─────────────────┐      ┌─────────────────┐
│ D1: y=0,1,11    │      │ D3: y=0,1,11    │
│ D2: y=5,6,7     │      │ D4: y=2,9,10    │
│                 │      │ D5: y=3,4,8     │
│                 │      │ D6: y=5,6,7     │
└─────────────────┘      └─────────────────┘
```

## Key Insight: NOC Directionality

**This is the most important concept to understand.**

NOCs are strictly unidirectional:
- **NOC0** can ONLY move +X (right) and +Y (down)
- **NOC1** can ONLY move -X (left) and -Y (up)

To reach a destination in the "wrong" direction, packets must wrap around the entire torus.

### Path Length Example

From (1,1) to (7,3):

| NOC | X hops | Y hops | Total | Wraps |
|-----|--------|--------|-------|-------|
| NOC0 | 6 (direct) | 2 (direct) | **8** | 0 |
| NOC1 | 4 (wrap) | 10 (wrap) | **14** | 2 |

NOC0 is clearly better for this route because the destination is down and to the right.

### When to Use Each NOC

| Destination relative to source | Best NOC |
|-------------------------------|----------|
| Right and/or Down | NOC0 |
| Left and/or Up | NOC1 |
| Far right (near wrap) | NOC1 might be shorter! |
| Far down (near wrap) | NOC1 might be shorter! |

## Using the Visualizer

### Views
- **Torus**: 3D torus showing true topology with wrap-around
- **Flat**: 2D grid view (easier to see coordinates)

### Adding Paths
1. Select which NOC to add paths to (NOC0 or NOC1)
2. Click a node to set the **source**
3. Click another node to set the **destination**
4. Path is automatically added

### Path Management
- **Hover** over a path in the list to highlight it
- **→0 / →1** buttons switch a path to the other NOC
- **×** removes a path
- Green switch button = switching saves hops
- Red switch button = switching costs more hops

### Animation
Press **Space** or click **Animate** to see packets flowing along paths.

### Keyboard Shortcuts
| Key | Action |
|-----|--------|
| `1` | Select NOC0 |
| `2` | Select NOC1 |
| `Space` | Toggle animation |
| `Esc` | Cancel pending selection |
| `Drag` | Rotate view |
| `Scroll` | Zoom |

## Presets

### Dual Mcast
Demonstrates independent multicast operations on each NOC:
- NOC0: Horizontal broadcast (+X)
- NOC1: Vertical broadcast (-Y)

### Bidirect
A ring topology using both NOCs for full-duplex communication:
- NOC0: Clockwise (right → down)
- NOC1: Counter-clockwise (left → up)

### Pipeline
Forward data flow with return path:
- NOC0: Stage1 → Stage2 → Stage3
- NOC1: Results return path

### Naive DRAM
**Anti-pattern**: All cores read via NOC0 only.

Problems:
- Right DRAM → Left cores requires long wrap paths
- NOC1 sits completely idle
- Only 50% of available bandwidth used

### Optimal DRAM
**Best practice**: Each DRAM group uses both NOCs.

```
LEFT DRAM (x=0):
  NOC0 (+X) → cores x=1,2  (1-2 hops direct)
  NOC1 (-X) → cores x=8,9  (1-2 hops via wrap!)

RIGHT DRAM (x=5):
  NOC0 (+X) → cores x=6,7  (1-2 hops direct)
  NOC1 (-X) → cores x=3,4  (1-2 hops direct)
```

**Key insight**: Left DRAM can efficiently reach far-right cores (x=8,9) via NOC1 wrapping:
- `x=0 → x=9` via NOC1 = **1 hop** (wraps 0→9)
- `x=0 → x=8` via NOC1 = **2 hops** (wraps 0→9→8)

This is shorter than NOC0 which would need 9 or 8 hops!

## Statistics

The visualizer shows:

| Metric | Description |
|--------|-------------|
| **NOC0/NOC1 Total Hops** | Sum of all path lengths per NOC |
| **Total Paths** | Number of communication paths |
| **Avg Hops/Path** | Efficiency metric (green ≤4, yellow ≤7, red >7) |
| **Max Hops** | Critical path length |
| **Bandwidth** | 2x if both NOCs used, 1x if single NOC |

## Best Practices for NOC Usage

### 1. Use Both NOCs
Wormhole has two independent NOCs. Using only one wastes 50% of available bandwidth.

### 2. Follow Natural Directions
- Data flowing right/down → NOC0
- Data flowing left/up → NOC1

### 3. Consider Wrap-Around
Sometimes wrapping around the torus is shorter than going the "direct" way:
- `x=0 → x=9` via NOC1 (wrap) = 1 hop
- `x=0 → x=9` via NOC0 (direct) = 9 hops

### 4. Balance DRAM Access
For interleaved DRAM operations:
- Left cores read from left DRAM via NOC0
- Right cores read from right DRAM via NOC0
- Far cores use NOC1 wrap-around

### 5. Separate Read/Write Traffic
- DRAM reads on one NOC
- DRAM writes on the other NOC
- Prevents contention

## Implementation Notes

When implementing kernels:

```cpp
// Choose NOC based on data flow direction
constexpr auto read_noc = NOC_0;   // Reading from lower coords
constexpr auto write_noc = NOC_1;  // Writing to higher coords

// Or dynamically based on relative positions
auto noc = (dst_x >= src_x && dst_y >= src_y) ? NOC_0 : NOC_1;
```

## References

- [Programming Tenstorrent Processors](https://martinlwx.github.io/en/programming-tenstorrent-processors/)
- [Memory on Tenstorrent](https://martinlwx.github.io/en/memory-on-tenstorrent/)
- TT-Metalium Documentation