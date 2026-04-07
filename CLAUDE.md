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
   - Internal separator lines (black, 1px) - top and bottom outer boundaries are not drawn
   - Timeline arrows: Material layer uses blue (#4A86E8), Team layer uses purple (#9f45fc)
   - Left-side labels: Blue vertical pill (Material) or purple vertical pill (Team)

3. **Business Nodes** - Two visual styles based on `styleType`:
   - `plan` (timeline layers): White rounded rectangle (rx=6, ry=6) with light gray border (#e3e4e6) + circle marker (layer-colored) + optional date label
   - `event` (intermediate lanes): Gray rounded rectangle (80x44px) with custom markup containing two text elements - date (gray #666, 9px) at refY:12, description (black #333, 11px, bold) at refY:28

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
    type: 'timeline' | 'lane'; // timeline=blue axis layer
  }[];
  nodes: {
    id: string;
    layerId: string;           // Maps to layers[].id
    date: string;              // Date for horizontal positioning (YYYY-MM-DD)
    label: string;
    styleType: 'plan' | 'event';
  }[];
  links: { source: string; target: string; }[];
}
```

## Key Implementation Details

- **Collision Avoidance**: Nodes with the same `date` are vertically aligned across layers. Within the same layer, nodes with the same date are horizontally offset to prevent overlap.

- **Graph Reset Strategy**: The component uses `graph.clearCells()` followed by full re-render on data changes. This is intentional for simplicity given the small data size.

- **Z-Index Layers**: Background lines (1), dependency edges (2), nodes/markers (5), labels (10)

- **X6 Configuration**: `interacting: false` (read-only), `panning: true`, `mousewheel` with Ctrl modifier

## File Structure

- `src/components/FlowTimeline.vue` - Core X6 graph component
- `src/App.vue` - Entry point with mock data
- `SPEC.md` - Data schema specification
