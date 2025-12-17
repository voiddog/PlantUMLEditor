# PlantUML Editor Project Context

## Overview
This is a modern PlantUML editor built with Vue 3 and Vite, featuring a "Liquid Glass" UI design. It allows users to manage multiple PlantUML diagrams simultaneously via tabs, with real-time (debounced) preview.

## Tech Stack
- **Framework:** Vue 3 (Composition API, `<script setup>`)
- **Build Tool:** Vite
- **Language:** JavaScript (ES Modules)
- **Styling:** SCSS, CSS Variables for theming
- **Editor:** `vue-codemirror` (CodeMirror 6)
- **Diagram Generation:** `plantuml-encoder` (Encodes text to PlantUML server URL)
- **Preview Interaction:** `panzoom` (Pan and Zoom functionality)

## Key Files
- `src/App.vue`: Main application logic. Handles state (files, tabs), editor binding, image URL generation, layout resizing, and pan/zoom interactions.
- `src/style.scss`: Global styles defining the "Liquid Glass" theme (variables, background, typography).
- `package.json`: Dependencies and scripts.

## Implementation Details
- **Multi-Tab State Management:** 
  - Uses `localStorage` to persist an array of file objects (`{ id, name, content }`).
  - Supports adding, deleting, and renaming diagrams (double-click tab to rename).
  - Remembers the last active tab and editor width.
- **Debounce:** Updates to the preview are debounced by 500ms to prevent excessive requests to the PlantUML server.
- **Preview:** The encoded string is appended to `https://www.plantuml.com/plantuml/svg/` to fetch the SVG.
- **Zoom/Pan:** 
  - The `panzoom` library is initialized on the `.img-wrapper` element.
  - **Zoom Interaction:** Double-clicking the preview area zooms in (default behavior restored).
  - **Unrestricted Panning:** Default bounds are disabled to allow free movement of the diagram.
- **Syntax Highlighting:** Achieved using `@codemirror/lang-java` for CodeMirror.
- **Resizable Layout:** Implemented with a custom draggable resizer between the editor and preview panes.

## Recent Changes
- **Feature:** Added multi-tab support with CRUD operations for diagrams.
- **Refactor:** Reverted "Double-click to Fit", restored default "Double-click to Zoom".
- **Refactor:** Removed experimental "Liquid Glass Cursor" component.
- **Refactor:** Fixed preview layout constraints to allow full panning.

## Future Improvements
- Add support for other PlantUML server endpoints.
- Add export to PNG/SVG buttons.
