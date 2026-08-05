# Wind Tunnel — Manual Save Build

This build performs **no automatic saving**.

State is written to browser `localStorage` only when **Save state** is clicked.

- **Save state** — manually stores the current configuration.
- **Load state** — restores the most recently saved configuration.
- **Clear saved state** — deletes the stored configuration.

There is no five-second timer, unload save, debounced save, or change-detection save.

The saved configuration includes geometry, imported SVG bodies, positions, size,
rotation, flip state, freehand obstacles, speed, visualization mode, ground plane,
detail setting, tool, selection, and pause/run state.

The live GPU fluid and smoke fields are not serialized. They restart and redevelop
after a saved configuration is loaded.
