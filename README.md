# JEV // ARCADE

A single-file browser arcade where JEV controls a live Snake game and exposes the reasoning
inputs, decisions, confidence, latency, and safety interventions.

## Run

No build step or dependencies are required.

```sh
python3 -m http.server 8000
```

Open [http://localhost:8000/arcade.html](http://localhost:8000/arcade.html), choose a brain,
and press **START ARCADE**. Local autopilot works without a key. Stop the server with `Ctrl-C`.

## Snake

Snake runs on a 25 x 25 grid. The body grows when it eats food. A wall collision or collision
with the body ends the current run and resets the board.

The decision space is relative to the current heading:

- `straight`: keep the current heading
- `left`: turn 90 degrees left
- `right`: turn 90 degrees right
- `uturn_left` and `uturn_right`: 180-degree turns, normally fatal because the neck occupies the destination

The game clock continues while a remote brain is thinking. Decisions are queued ahead of
application, so network latency does not pause the simulation. The safety net checks the
selected move again at application time and can replace a fatal answer with the local pilot's
best safe move.

The panel reports score, deaths, moves, best score, turns, safety-net interventions, choice
probabilities, confidence, latency, and an event log.

## JEV implementation

The implementation lives in `arcade.html` and is intentionally dependency-free.

### Decision flow

1. `snakePilot()` evaluates candidate turns with collision simulation, breadth-first search,
   flood-fill area checks, and tail-reachability checks.
2. `sBuildQuery()` serializes the future Snake state and the pilot recommendation.
3. `brainAsk()` sends the query to the selected brain and normalizes the returned choice.
4. `snakePrimeQueue()` fills the initial decision queue before the game starts.
5. `snakeDecide()` keeps the queue filled while the game runs.
6. `snakeClock()` applies exactly one queued relative turn per tick, validates it, advances the
   board, and records the result.

### Brain modes

- **Local autopilot**: uses the deterministic `snakePilot()` implementation. It needs no network
  connection or API key.
- **JEV / system one**: sends structured state to `https://openrouter.ai/api/v1/systemone` using
  the `jev-latest` model.
- **OpenRouter model**: sends the same decision context to the OpenRouter chat completions API.

JEV and chat responses are normalized to one of the five turn keys. Probability data is cleaned
and normalized before it is shown in the UI. If system one fails, the page records the error and
does not silently invent a remote result.

## Configuration

- `GRID` and `CELL` define the board dimensions and rendered cell size.
- `QUEUE_TARGET` controls how many future decisions are buffered.
- The speed slider changes the Snake tick interval in milliseconds.
- The safety-net checkbox controls whether fatal remote choices may be overridden.

API keys are held in page memory and sent only to the selected OpenRouter endpoint. Do not use
this demo as a secure secret-management system.

## Troubleshooting

- A blank page usually means the file was opened with a browser restriction; serve it over the
  local HTTP server shown above.
- For remote modes, verify the API key, model access, and browser network permissions.
- Use Local autopilot to test gameplay without any external service.
