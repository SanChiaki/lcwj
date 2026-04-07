# Project: Flow Timeline Visualization

## Overview
This project is a Vue 3-based web application that renders a business process timeline using **AntV X6**. It is designed to visualize the flow of materials through various organizational layers (Material Planning, Warehouse/Logistics, and Front-line Teams).

The diagram features:
- **Swimlane Structure:** Organizational layers (lanes) for different departments.
- **Timeline Integration:** Top and bottom layers function as interactive timelines with date markers and business milestones.
- **Dynamic Routing:** Dependency links between business nodes use the Manhattan routing algorithm with obstacle avoidance to prevent overlapping with nodes.
- **Collision Avoidance:** Automatic horizontal shifting of nodes within the same lane to ensure readability even when time indices are close.

## Technical Stack
- **Framework:** Vue 3 (Composition API)
- **Diagramming:** AntV X6 (v2.17.1)
- **Build Tool:** Vite
- **Styling:** Scoped CSS

## Key Commands
- **Install Dependencies:** `npm install`
- **Development Server:** `npm run dev`
- **Build for Production:** `npm run build`
- **Preview Production Build:** `npm run preview`

## Directory Structure
- `src/components/FlowTimeline.vue`: The core component responsible for mapping business data to the X6 graph coordinate system and rendering the visual elements.
- `src/App.vue`: The entry point component that provides mock data and configuration to the timeline.
- `SPEC.md`: Technical specification and data schema definition.

## Development Conventions
- **Data-Driven Rendering:** The graph is rendered purely based on the input JSON data structure (`layers`, `nodes`, `links`, `timeline`).
- **Coordinate System:**
  - **X-axis:** Calculated based on `timeIndex` * `columnWidth`, with automatic gap adjustment for collisions.
  - **Y-axis:** Calculated based on `layerIndex` * `rowHeight`, with vertical centering within lanes.
- **Component Design:** The `FlowTimeline` component uses a `watch` on its `data` prop to perform full re-renders (clear and redraw) when data changes.

## Data Schema
Refer to `SPEC.md` for the detailed TypeScript interface of the input data. Key properties include:
- `timeIndex`: A float value (e.g., 1.5) used to position nodes precisely between main timeline markers.
- `styleType`: Distinguishes between `plan` (blue border milestones) and `event` (gray logistics events).
