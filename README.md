<div align="center">

# nova-ufast-agent

**A browser automaton that indexes what it sees, decides what it does, and types only the words.**

`pip install nova-ufast-agent`

[![PyPI - Version](https://img.shields.io/pypi/v/nova-ufast-agent?color=6c8cff&label=version)](https://pypi.org/project/nova-ufast-agent/)
[![PyPI - Python](https://img.shields.io/pypi/pyversions/nova-ufast-agent?color=37d5b9)](https://pypi.org/project/nova-ufast-agent/)
[![PyPI - License](https://img.shields.io/pypi/l/nova-ufast-agent?color=brightgreen)](https://github.com/Tanmay-Somani/nova-ufast-agent/blob/main/LICENSE)
[![Tests](https://img.shields.io/badge/tests-31%20passing-brightgreen)]()

</div>

---

## Table of contents

- [What it does](#what-it-does)
- [Highlights](#highlights)
- [How decisions are made](#how-decisions-are-made)
- [Requirements](#requirements)
- [Install](#install)
- [Quick start](#quick-start)
- [Using the library](#using-the-library)
- [Configuration](#configuration)
- [Operations](#operations)
- [API reference](#api-reference)
- [Examples](#examples)
- [Project layout](#project-layout)
- [Safety model](#safety-model)
- [Development](#development)
- [Troubleshooting](#troubleshooting)
- [Known limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License and credits](#license-and-credits)

---

## What it does

Point it at a URL, give it a goal in plain language, and it drives a real Chrome
session until the goal is visibly satisfied (or it has proven it can't).

Most "browser agents" generate low-level commands — click `(x, y)`, fill selector
`#email`, run JavaScript — which is how models leak state, hallucinate selectors,
and wreck pages. **nova-ufast-agent inverts that model.** The agent reads the page
into a numbered list of concrete, observable controls, then asks a decision model
to pick *which numbered item to touch* and *with which operation*. The model never
sees an empty action vocabulary and never invents a locator.

The only generated content is text. When the chosen operation is `TYPE_TEXT`, a
small, cheap LLM produces the string to insert; everything else is executed by the
library against the DOM node it already observed.

## Highlights

- **One decision, two answers, one round trip.** Operation and element-target
  heads share a single observed state and a single network request.
- **Code-owned execution.** Observed node references, not model-generated
  selectors, coordinates, shell commands, or inline JavaScript.
- **Safety guards on every input.** Freshness checks, geometry resolution,
  occlusion/replacement detection, and disabled/readonly rejection run right
  before anything mutates the page.
- **No screenshots in the hot path.** The model consumes structured state;
  the built-in inspector opts into screenshots, the loop does not.
- **Structured snapshots in one browser call.** Visible controls, names, values,
  and page text are read atomically and sent onward.
- **Bounded, retry-safe.** Decisions are consumed exactly once, mutations are
  never replayed, and interrupted text generation can be reused only when its
  full input context is byte-identical.
- **Works offline.** A fixture harness and a fully offline test suite let you
  develop without paid model calls.

## How decisions are made

Each iteration produces a fresh element table, indexed from `[1]`:

```text
[1] button    Change ticket type · Round trip
[2] combobox  Where from?        · San Francisco
[3] combobox  Where to?          · empty
[4] textbox   Departure          · empty
...
```

The agent asks one question about the *operation* and one about the matching
*target* (only the operation that gets selected can actually execute):

```text
                       one decision request
                      ┌───────────────────────────┐
page → element table → operation                  │
                      │ click_target              │
                      │ type_text_target          │
                      │ select_target, if present │
                      └─────────────┬─────────────┘
                          the matching target only
                                    │
                     CLICK [7] ─────┤──→ browser
                 TYPE_TEXT [3] ─────┘
                           │
                           ↓
                    text LLM → value → browser
```

Target questions are speculative fan-out: if the model says `CLICK`, only the
`click_target` head can influence the action. Extra heads can't fire side effects.

## Requirements

- Python **3.12 or newer**
- **Chrome** installed and able to open a remote-debugging port (via
  [browser-harness](https://github.com/browser-use/browser-harness))
- A **TypeSafe** API key for decision making
- A text-model API key (OpenAI-compatible endpoint) only if `TYPE_TEXT` will be used
- Linux, macOS, or Windows

## Install

```bash
# PyPI
pip install nova-ufast-agent        # or: uv add nova-ufast-agent

# Latest source
git clone https://github.com/Tanmay-Somani/nova-ufast-agent.git
cd nova-ufast-agent
uv sync
```

## Quick start

Set your keys (copy `.env.example` to `.env` and fill it in):

```bash
cp .env.example .env
```

Start the local inspector:

```bash
uv run nova-ufast
```

Open **http://127.0.0.1:8766**, pick a scenario, and click **Start demo**. You'll
see the numbered elements, the operation/target probabilities, and a running
decision trail. Use **Choose next** to pause before each execution and inspect
what the model picked.

If Chrome doesn't attach, run `uv run browser-harness --doctor` and allow remote
debugging when prompted.

## Using the library

The entry point is the [`Agent`](#agent) context manager; iterate its `run()`
generator to receive a state snapshot after every step:

```python
from nova_ufast_agent import Agent

with Agent(
    "https://www.google.com/travel/flights?hl=en",
    "Find one-way flights from Zurich to London on September 20, 2026, "
    "for one adult in economy. Stop when matching flight options are visible.",
) as agent:
    for state in agent.run():
        print(f"{state['elapsed_ms']} ms  status={state['status']}")
```

Each yielded state is a plain dict with `status`, `page`, `decision`, `history`,
and more — see [API reference](#api-reference).

A single goal string is the normal case, but `goals` also accepts a list to
create an ordered plan:

```python
goals = [
    "Open the search page.",
    "Filter to results published this year.",
    "Open the top result.",
]
with Agent("https://example.com", goals, screenshots=True) as agent:
    for state in agent.run():
        ...
```

## Configuration

Everything is configured through environment variables (optionally via a `.env`
file in the working directory). Secrets never ship in code.

| Variable | Required | Default | Purpose |
| --- | --- | --- | --- |
| `TYPESAFE_API_KEY` | Yes | – | Decision model (operation + target heads) |
| `TYPESAFE_MODEL` | No | `jev-latest` | Decision model identifier |
| `TEXT_MODEL_API_KEY` | For `TYPE_TEXT` | – | OpenAI-compatible text helper |
| `TEXT_MODEL` | No | `deepseek-chat` | Text helper model |
| `TEXT_MODEL_BASE_URL` | No | `https://api.deepseek.com/v1` | Text helper endpoint |
| `TEXT_MODEL_REASONING` | No | – | Set `none` to disable reasoning on compatible providers |
| `TYPESAFE_DEMO_PORT` | No | `8766` | Port for the local inspector |

## Operations

| Operation | Meaning |
| --- | --- |
| `CLICK` | Press an observed element (button, link, option, calendar day, …) |
| `TYPE_TEXT` | Replace a field's value with a string generated by the text LLM |
| `SELECT` | Choose an observed option of a native `<select>` |
| `SCROLL_UP` / `SCROLL_DOWN` | Scroll the viewport |
| `WAIT` | Give the page a moment when the needed control isn't ready |
| `DONE` | Declare the goal visibly satisfied |
| `BLOCKED` | No supported operation can make progress |

Only supported operations and their compatible targets are offered to the model
on any given observation.

## API reference

### `Agent`

```python
Agent(url: str, goals: str | list[str], *, record_dir: str | None = None, screenshots: bool = False)
```

- `url` — page to open.
- `goals` — one goal, or an ordered list forming a plan.
- `record_dir` — if set, frames are written here as numbered JPEGs (implies
  screenshots).
- `screenshots` — capture and attach screenshots to each observation (off by
  default to keep the decision loop lean).

Methods:

- `.run()` — generator of state dicts; yields once per executed step until
  `status` is `done` or `blocked`.
- `.snapshot()` — the current state dict.
- `.command(name, body=None)` — low-level step control (`predict`, `act`, `tick`);
  used by the inspector.
- `.close()` — closes the owned browser tab.
- Context-manager compatible (`with`).

State dict contains at least: `status`, `page`, `decision`, `history`,
`decisions`, `goal`, `plan`, `plan_index`, `elapsed_ms`, `text_calls`, `record`,
and `elements` (the flattened element table). `page` embeds `url`, `title`,
`text`, `actions` (observed controls with `id`, `kind`, `label`, `node`,
`rect`, …), `marker`, `page_key`, and `guards`.

### `Browser`

```python
Browser(url: str)
```

Connects a single CDP session to an owned Chrome tab (created in the background
so the user's visible tab isn't swapped), renders it with focus emulation, and
exposes `observe()`, `act()`, `fresh()`, `call()`, `evaluate()`, and `close()`.
Raises `nova_ufast_agent.browser.StalePage` whenever a decision no longer
matches the observed page.

### Model helpers (`nova_ufast_agent.model`)

- `action_space(actions)` — turns raw observed actions into indexed elements,
  per-operation target maps, and the fixed control set.
- `choose(state, goal, history)` — one decision round trip; returns the selected
  operation, target, confidence, per-choice probabilities, and usage.
- `field_context(goal, action, page, history)` / `field_text(context)` — build
  the text-helper input and validate its JSON-only output.

## Examples

Both examples live in `examples/` and call real paid APIs:

```bash
# Run any URL/goal
uv run --env-file .env python examples/run.py \
  --url https://en.wikipedia.org/wiki/Main_Page \
  --goal 'Find and open the Wikipedia article about Gödel’s incompleteness theorems.'

# Live Google Flights search with an independent result check (never books)
uv run --env-file .env python examples/flights.py --keep-open
```

## Project layout

```text
nova_ufast_agent/
├── agent.py        # the complete decision/execute loop + text-helper handoff
├── browser.py      # CDP session, snapshots, freshness/occlusion guards, input
├── model.py        # operation + target heads, decision validation, text helper
├── questions.py    # model instructions and step budget (MAX_STEPS = 60)
├── snapshot.js     # atomic DOM reader producing indexed, guarded actions
├── demo.py         # loopback-only local inspector server
└── static/         # inspector frontend (HTML/CSS/JS + offline fixtures)
examples/           # run.py (any goal), flights.py (verified Flights search)
tests/              # offline contract tests – no paid APIs
```

## Safety model

1. **Everything executes from an observed node.** Model output is an index into
   the observed element table — never a selector, offset, script, or shell line.
2. **Re-verify before touching.** Prior to input, the executor re-checks
   document freshness, element connectivity/disabled state, geometry, and
   center-hit coverage (`elementFromPoint`).
3. **Protect real fields.** Disabled, readonly, and `aria-readonly` fields are
   rejected.
4. **One chance per mutation.** A decision is burned before any work; a stale
   retry can never double-fire an action.
5. **Text is gated.** `TYPE_TEXT` requires validated JSON from a permissioned
   helper — free-text from the decision model can't reach the keyboard path.
6. **`DONE` is a claim, not a proof.** The loop stops, but verifying the real
   outcome is on you (see `examples/flights.py::verify`).

## Development

```bash
uv sync
uv run ruff check .
uv run pytest
node --check nova_ufast_agent/static/app.js
node --check nova_ufast_agent/snapshot.js
uv build
```

The test suite runs fully offline — model calls are mocked, no API keys needed.

## Troubleshooting

| Symptom | Likely fix |
| --- | --- |
| Chrome won't connect | `uv run browser-harness --doctor`; allow remote debugging when Chrome asks |
| `403` from the inspector | The demo server only accepts locals requests; don't proxy or expose port 8766 |
| `TypeError: ... __init__() got an unexpected keyword argument` | Old build cached; reinstall `nova-ufast-agent` (or `uv sync`) |
| `TEXT_MODEL_API_KEY` errors only when typing | Expected — it's only needed for `TYPE_TEXT` |
| Demo port busy | Set `TYPESAFE_DEMO_PORT` to a free port |

## Known limitations

- DOM reading covers common HTML/ARIA controls; it is not the full
  accessible-name spec.
- Shadow roots, iframes, canvas, uploads, new pop-up tabs, nested scrollers, and
  arbitrary keyboard widgets are unsupported.
- Owned tabs share the existing Chrome profile.
- `DONE` decisions still require independent outcome verification.

## Roadmap

- [x] Rebranded fork with clean package boundary (`nova_ufast_agent`)
- [x] Repackaged and vetted for PyPI (wheel + sdist, py3.12+)
- [x] Rebuilt inspector frontend (dark theme, animated overlays)
- [ ] Pluggable decision/helper adapters (OpenAI-compatible choice endpoints)
- [ ] Shadow-root / frame traversal in the DOM reader
- [ ] Optional headless mode
- [ ] GitHub Actions CI (lint + tests + publish on tag)

## Contributing

PRs, issues, and ideas are welcome. Keep the loop small — page → indexed
elements → operation + target → execution — and never let the model emit
executable instructions. Read `AGENTS.md` before editing. Tests must stay
offline and free of paid API calls.

## License and credits

MIT — see [LICENSE](LICENSE). This project started as a rebranded, repackaged
derivative of Browser Use's [jev-ultrafast](https://github.com/browser-use/jev-ultrafast)
and is grateful to its authors for the design.