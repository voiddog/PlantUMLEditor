# PlantUML Editor Project Context

## Overview
This is a modern PlantUML editor built with Vue 3 and Vite, featuring a "Liquid Glass" UI design. It allows users to write PlantUML code and see a real-time (debounced) preview of the diagram.

## Tech Stack
- **Framework:** Vue 3 (Composition API, `<script setup>`)
- **Build Tool:** Vite
- **Language:** JavaScript (ES Modules)
- **Styling:** SCSS, CSS Variables for theming
- **Editor:** `vue-codemirror` (CodeMirror 6)
- **Diagram Generation:** `plantuml-encoder` (Encodes text to PlantUML server URL)
- **Preview Interaction:** `panzoom` (Pan and Zoom functionality)

## Key Files
- `src/App.vue`: Main application logic. Handles state, editor binding, image URL generation, and layout.
- `src/style.scss`: Global styles defining the "Liquid Glass" theme (variables, background, typography).
- `package.json`: Dependencies and scripts.

## Implementation Details
- **State Management:** Uses Vue `ref` for code and image URL. Code and editor width are persisted to `localStorage`.
- **Debounce:** Updates to the preview are debounced by 500ms to prevent excessive requests to the PlantUML server.
- **Preview:** The encoded string is appended to `https://www.plantuml.com/plantuml/svg/` to fetch the SVG.
- **Zoom/Pan:** The `panzoom` library is initialized on the `.img-wrapper` element within the preview pane.
- **Syntax Highlighting:** Achieved using `@codemirror/lang-java` for CodeMirror, providing general code highlighting.
- **Resizable Layout:** Implemented with a custom draggable resizer, allowing users to adjust the width of the editor and preview panes. The chosen width is saved to `localStorage`.
- **Error Handling:** Errors during encoding or image loading are logged to the browser console.

## Future Improvements
- Add local storage persistence.
- Add support for other PlantUML server endpoints.
- Add export to PNG/SVG buttons.
