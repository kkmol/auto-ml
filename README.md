# auto-ml (Model Desk)

Planning documents and UI mocks for Model Desk, the internal auto-ML platform.

## Planning documents

- [docs/platform-vision.md](docs/platform-vision.md): platform vision and build spec
- [docs/build-spec-hackathon.md](docs/build-spec-hackathon.md): hackathon build spec (six contracts, three workstreams)

These are markdown exports of the Claude Doc "Model Desk — platform vision and build spec". Embedded diagrams are not included in the export.

## UI mocks

Source: Claude design canvas "Internal Auto-ML — chat UX":
https://claude.ai/artifact/HesUzLJDsxfqTTBsMqQwTF

The v3 ("agentic") screens are the baseline layout and are in `mocks/v3/`:

- `V3Home.dc.html`: Home, agent inbox sorted by what needs you
- `V3Agent.dc.html`: agent at work (interactive demo)
- `V3Live.dc.html`: live model, agent as caretaker
- `V3Settings.dc.html`: settings (interactive)
- `V3Composer.dc.html`: composer spec, `/` commands and Shift+Tab modes

`mocks/canvas.json` is the canvas layout and notes for all three pages. It also lists the v1 and v2 boards, which are not in this repo yet; they remain in the canvas.

The `.dc.html` files are design-canvas components. They load `./support.js`, which comes from the canvas runtime and is not included here, so open them in the canvas rather than directly in a browser.
