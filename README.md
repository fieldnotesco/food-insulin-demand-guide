# Food Insulin Demand Ledger

A single-file HTML app for looking up, tallying, and estimating Food Insulin Demand
(FID) values — built for PCOS / metabolic health nutrition tracking around the
Food Insulin Index (FII) system from the University of Sydney's Brand-Miller / Bell
research group.

**Start here → [`CLAUDE.md`](./CLAUDE.md)** — full project context for picking up
development in Claude Code: current state, architecture, data model, and key decisions.

## Project structure

```
.
├── README.md                    ← you are here
├── CLAUDE.md                    ← main context file, read this first
├── DATA_SOURCES.md              ← where every number in the app came from
├── CALCULATOR_METHODOLOGY.md    ← the math behind the 3 calculator tabs, and why
└── insulin-demand-ledger.html   ← the app itself (open directly in a browser)
```

## Running it

No build step. Open `insulin-demand-ledger.html` directly in a browser, or serve
the folder with any static file server. The tally-persistence feature uses the
Claude artifact `window.storage` API when hosted as a Claude artifact, and falls
back to the browser's `localStorage` otherwise — everything works standalone.
