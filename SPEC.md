# Flow Timeline Component Specification

## 1. Overview
This component renders a business process timeline (Supply Chain/Logistics focus) using AntV X6 and Vue 3. It visualizes the flow of materials from planning to delivery across multiple organizational layers.

## 2. Technical Stack
- **Framework:** Vue 3 (Composition API)
- **Diagramming Library:** AntV X6 (v2.17.1)
- **Styling:** Vanilla CSS / Scoped CSS

## 3. Data Structure (Input Schema)

```typescript
interface FlowData {
  // Horizontal timeline labels (e.g., dates)
  timeline: string[]; 
  
  // Vertical layers/lanes
  layers: {
    id: string;
    label: string;
    type: 'timeline' | 'lane'; // timeline: blue background axis; lane: simple text
  }[];

  // Business nodes
  nodes: {
    id: string;
    layerId: string;           // Maps to layers[].id
    date: string;              // Date for horizontal positioning (YYYY-MM-DD)
    label: string;
    styleType: 'plan' | 'event'; // plan: blue border; event: gray rounded rect
  }[];

  // Dependency links
  links: {
    source: string;
    target: string;
  }[];
}
```

## 4. Visual Requirements
- **Layers:** Separated by horizontal lines with arrows pointing right.
- **Headers:** "物料" (Top) and "队伍" (Bottom) with vertical blue background labels.
- **Nodes:**
  - `plan`: Rectangular, white background, thick blue border.
  - `event`: Rounded rectangle, light gray background, white text.
- **Connections:** Smooth curved blue lines with arrows.
- **Coordinate System:**
  - X = Based on `date` value (nodes with same date are vertically aligned)
  - Y = `PaddingTop + (layerIndex * RowHeight)`

## 5. Implementation Plan
1. Initialize Vue 3 project environment.
2. Install `@antv/x6@2.17.1`.
3. Create `FlowTimeline.vue` component.
4. Implement the mapping logic from `FlowData` to X6 Graph objects.
5. Add mock data and verify rendering.
