# PlantUML Editor

A modern, online PlantUML editor with a beautiful "Liquid Glass" interface.

## Features

- **Modern UI:** Glassmorphism design with backdrop blur and gradients.
- **Real-time Preview:** Automatically updates the diagram as you type (with debounce).
- **Interactive Preview:** Drag to pan and scroll/pinch to zoom the diagram.
- **Syntax Highlighting:** Code editor with line numbers and dark theme for improved readability.
- **Resizable Layout:** Drag the divider to adjust the size of the editor and preview panes.
- **Persistence:** Your code and layout preferences are saved locally in the browser.

## Prerequisites

- Node.js (v16+)
- npm

## Setup & Run

1.  **Install dependencies:**
    ```bash
    npm install
    ```

2.  **Start development server:**
    ```bash
    npm run dev
    ```

3.  Open your browser (usually at `http://localhost:5173`) to use the editor.

## Tech Stack

- Vue 3
- Vite
- CodeMirror
- PlantUML Encoder

## License

MIT
