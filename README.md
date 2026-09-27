# JEV Plays Snake

A single-page Snake experiment in which JEV selects the next relative turn while the game keeps
moving. The page makes the decision loop observable: it shows the model's choice, probabilities,
confidence, request latency, pilot recommendation, score, deaths, and safety-net interventions.

## Quick Start

The project has no build system, package manager, or runtime dependency.

```sh
python3 -m http.server 8000
```

Open [http://localhost:8000/arcade.html](http://localhost:8000/arcade.html), select a brain,
and press **START ARCADE**. The local autopilot needs no API key. Stop the server with `Ctrl-C`.

## Technology Stack

- **HTML5**: page structure, controls, modal configuration, and statistics panel.
- **CSS3**: the dark terminal-style interface, layout, status colors, and probability bars.
- **Vanilla JavaScript**: game state, decision orchestration, API requests, normalization, and
  safety checks. There are no frontend frameworks or third-party JavaScript packages.
- **HTML Canvas 2D**: renders the 25 x 25 Snake board, grid, food, body, and interpolation between
  ticks.
- **Browser Fetch API**: sends decision requests directly from the browser to OpenRouter.
- **`requestAnimationFrame` and timers**: animation rendering runs separately from the fixed Snake
  tick clock, while asynchronous brain requests refill the decision queue.

## Architecture

Everything is intentionally contained in `arcade.html`:

```text
Browser UI
  -> brainAsk()
       -> Local autopilot: snakePilot()
       -> JEV: OpenRouter System One / jev-latest
       -> Chat mode: OpenRouter chat completions / selected model
  -> normalized relative turn
  -> queued decision
  -> safety validation
  -> Snake tick and Canvas render
```

### Game layer

Snake occupies a 25 x 25 grid. The game advances every `sTick` milliseconds, which defaults to
160 ms and can be changed with the speed slider. The head moves continuously in its current
heading. A decision changes that heading once, then the snake continues forward.

The decision vocabulary is relative to the current heading:

- `straight`: keep moving forward
- `left`: turn 90 degrees left
- `right`: turn 90 degrees right
- `uturn_left` and `uturn_right`: reverse direction; these are normally fatal because the neck
  occupies the destination cell

Food grows the body and increases the score. Hitting a wall or the body resets the run and
increments the death counter.

### Deterministic pilot

`snakePilot()` is the local planning and safety layer. It evaluates candidate moves using:

- collision simulation with `simStep()`;
- breadth-first search for paths to food and the tail;
- flood-fill area estimates to avoid small traps;
- tail-reachability checks after eating;
- a tail-shadow strategy when a direct food path would close the snake into itself.

The pilot is also the local brain mode and provides a recommendation alongside remote model
answers. When the safety net is enabled, a queued move is checked again immediately before it is
applied. A fatal answer can be replaced by the pilot's safest available move.

### Decision queue

`snakePrimeQueue()` fills the initial queue before the game starts. `snakeDecide()` keeps the
queue at `QUEUE_TARGET` decisions while the simulation continues. This means the game does not
freeze during an API request and a response arriving late does not directly pause gameplay.

`snakeClock()` removes one relative turn from the queue per tick, validates it against the live
board, applies it, updates statistics, and schedules the next tick. `sRenderLoop()` uses Canvas
and `requestAnimationFrame` to draw the current state smoothly between game ticks.

## JEV and Model Sources

### JEV mode

The default JEV mode calls OpenRouter's System One endpoint:

```text
https://openrouter.ai/api/v1/systemone
```

The request identifies the model as **`jev-latest`** and sends structured state rather than a
raw screenshot. The state includes the grid size, head and food coordinates, body length,
heading, food distance, safe and unsafe turns, projected cells, recent turns, and the deterministic
pilot recommendation. The model returns a decision answer and may return probabilities and
confidence.

JEV is therefore an external hosted model accessed through OpenRouter; the model weights are not
stored in this repository. The repository contains the browser client, state serializer, response
normalizer, and game logic, not the model itself.

### OpenRouter chat mode

The optional chat mode calls:

```text
https://openrouter.ai/api/v1/chat/completions
```

The user supplies a provider/model identifier. The browser asks the selected model to finish with
a compact JSON object containing a choice and confidence. The client extracts and normalizes that
JSON before putting the choice into the queue.

### Local mode

Local mode never calls a network service. It uses `snakePilot()` directly, which makes it useful
for testing the game and UI without credentials or external model availability.

## AI Latency

The UI reports latency for each remote decision. It is measured in `brainAsk()` with
`performance.now()` from immediately before the request or local decision until the result is
normalized:

```js
const t0 = performance.now();
// request and response parsing
return { ...result, latency: Math.round(performance.now() - t0) };
```

For the current JEV setup, observed response latency is approximately **400 ms** in a typical
run. That is an empirical round-trip figure, not a guaranteed model speed: it includes browser
request overhead, network time, OpenRouter routing, model execution, and response parsing. It can
vary with location, traffic, model load, and provider conditions.

The queue is intentionally maintained ahead of the live Snake state. Consequently, the game can
continue ticking while the next JEV answer is being computed. The displayed latency is useful for
diagnostics, but it is not treated as the game clock.

Timeouts are set to 8 seconds for JEV System One and 9 seconds for chat mode. Errors are recorded
in the event log and shown in the status line.

## Configuration and Security

- `GRID` and `CELL` define the board dimensions and rendered cell size.
- `QUEUE_TARGET` controls how many future decisions are buffered.
- `sTick` controls the Snake tick interval; the UI slider changes it at runtime.
- The safety-net checkbox controls whether fatal remote choices may be overridden.

API keys are held only in page memory and sent to the selected OpenRouter endpoint. This is a
client-side demo, not a secure production secret-management system. Do not deploy it publicly with
privileged keys embedded in the page.

## Files

```text
arcade.html   Complete Snake game, UI, JEV client, pilot, and renderer
README.md     Project architecture and usage documentation
```

## Troubleshooting

- If the page does not load correctly, serve it through the local HTTP server instead of opening
  the file directly.
- Use Local autopilot to confirm the game works before debugging a remote model connection.
- For JEV or chat mode, verify the API key, model access, browser network permissions, and the
  status/event log.
