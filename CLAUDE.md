# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Vue 3-based supply chain timeline visualization using AntV X6. Renders a swimlane diagram showing material flow through organizational layers (Material Planning → Warehouse/Logistics → Front-line Teams).

## Key Commands

```bash
npm install    # Install dependencies
npm run dev    # Start dev server (Vite)
npm run build  # Production build
npm run preview # Preview production build
```

## Architecture

### Data Flow
1. `App.vue` provides `FlowData` object containing `layers`, `nodes`, and `links`
2. `FlowTimeline.vue` watches data changes and triggers full re-render via `renderFlow()`
3. Graph is cleared and rebuilt on every data change (not incremental)

### Rendering Pipeline (`FlowTimeline.vue`)

The `renderFlow()` function executes in 4 phases:

1. **Position Calculation** - Maps `date` to X coordinates with vertical alignment for same dates:
   - Collect unique dates, sort them, assign fixed X positions with `DATE_GAP` (130px)
   - Nodes with same `date` are vertically aligned across layers
   - Same-layer, same-date nodes are horizontally offset to prevent overlap

2. **Layer Structure** - Renders swimlane boundaries and timeline axes:
   - Internal separator lines: light gray dashed (`#d0d0d0`, `strokeDasharray: '5,5'`) — rendered as **edges** (not rect nodes), see pitfall below
   - Timeline arrows: Material layer uses blue (#4A86E8), Team layer uses purple (#9f45fc)
   - Left-side labels: Blue vertical pill (Material/timeline) or gray plain text (lane)

3. **Business Nodes** - Two visual styles based on `styleType`:
   - `plan` (timeline layers): White rounded rectangle (120×60px, rx=6, ry=6), light gray border (#e3e4e6), custom markup with 3 text rows — orderNo (gray #999, 10px, refY:12), label (bold #333, 13px, refY:30), material (gray #888, 10px, refY:48)
   - `event` (lane nodes): Gray rounded rectangle (80–120×60px), custom markup with 3 text rows — orderNo (#888, 9px, refY:12), date (#666, 9px, refY:28), label (bold #333, 11px, refY:45)

4. **Dependency Links** - Manhattan routing with obstacle avoidance:
   - Router: `manhattan` with 20px padding
   - Connector: `smooth` with 20px radius
   - Blue lines with arrow markers

### Coordinate System

- **X-axis**: Date-based, nodes with same `date` are vertically aligned, compact spacing with `DATE_GAP: 130`
- **Y-axis**: Layer-based, `startY + layerIndex * rowHeight`, vertically centered within lane
- **Config**: `columnWidth: 200`, `rowHeight: 120`, `startX: 150`, `startY: 60`, `minGap: 60`

### Data Schema (FlowData)

```typescript
{
  layers: {
    id: string;
    label: string;
    type: 'timeline' | 'lane'; // timeline=colored axis layer, lane=gray label
  }[];
  nodes: {
    id: string;
    layerId: string;           // Maps to layers[].id
    date: string;              // Date for horizontal positioning (YYYY-MM-DD)
    label: string;
    styleType: 'plan' | 'event';
    orderNo?: string;          // Order/document number (shown on all node types)
    material?: string;         // Material description (plan nodes only)
  }[];
  links: { source: string; target: string; }[];
}
```

## Key Implementation Details

- **Collision Avoidance**: Nodes with the same `date` are vertically aligned across layers. Within the same layer, nodes with the same date are horizontally offset to prevent overlap.

- **Graph Reset Strategy**: The component uses `graph.clearCells()` followed by full re-render on data changes. This is intentional for simplicity given the small data size.

- **Z-Index Layers**: Background lines (1), dependency edges (2), nodes/markers (5), labels (10)

- **X6 Configuration**: `interacting: false` (read-only), `panning: true`, `mousewheel` with Ctrl modifier

## Known Pitfalls

- **Do NOT use `rect` nodes for dashed separator lines.** A 1px-height rect with `stroke` renders the stroke on both the top and bottom edge, producing a visually doubled line. Use `graph.addEdge()` with `strokeDasharray` instead:
  ```javascript
  graph.addEdge({
    source: { x: startX, y: y },
    target: { x: endX, y: y },
    attrs: { line: { stroke: '#d0d0d0', strokeWidth: 1, strokeDasharray: '5,5', targetMarker: null } },
    zIndex: 1
  });
  ```

## File Structure

- `src/components/FlowTimeline.vue` - Core X6 graph component
- `src/App.vue` - Entry point with mock data
- `SPEC.md` - Data schema specification
