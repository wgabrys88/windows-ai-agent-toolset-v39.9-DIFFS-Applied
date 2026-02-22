# Stateless Vision-Loop Desktop Agent

This repository implements a minimal agentic control loop for a Windows desktop driven by a vision-language model (VLM).
The system is intentionally "stateless" at the planner level: the VLM plans from (1) a short textual story it tells itself
and (2) visual annotations burned into the screenshot it receives every turn.

Files:
- main.py     Python engine: executes actions, captures screenshots, hosts a local HTTP UI, calls the VLM API.
- panel.html  Browser UI: renders screenshots, draws overlays (heatmaps, labels), exports annotated screenshots back to Python.
- config.py   All runtime configuration (HTTP, VLM, capture, execution, UI overlays, boot injection, logging layout).

Requirements:
- Windows 11
- Python 3.13+
- Google Chrome (latest) recommended for panel.html rendering
- No third-party Python packages (stdlib + ctypes + WinAPI only)


## Philosophy: the "AI" is the story + visual feedback

The engine is a deterministic executor and logger. The model is guided primarily by:
1) A short text "observation" that it produces each turn (its internal story).
2) The annotated screenshot it receives, which contains visual feedback about what it has done.

Key design points:
- The screenshot is captured AFTER actions are executed.
- The browser overlays an orange heatmap for the actions from that same turn.
- Optionally, the panel keeps a short fading trail of recent action heatmaps across multiple turns.
- This turns "I did something" into a visual fact embedded into the next model input, even when the UI change is subtle.

The system avoids explicit Python-side loop detection or repetition counting. The model is expected to self-correct by
seeing its recent actions in the annotated image and narrating the next step in the observation text.


## High-level loop

One turn corresponds to one VLM response (JSON), one physical execution, one screenshot, one annotation, and one VLM call.

### ASCII diagram (single turn):

```
  +-------------------+
  | VLM (JSON output) |
  +---------+---------+
            |
            v
  +---------------------------+
  | Python: parse JSON        |
  | - observation             |
  | - bboxes (optional)       |
  | - actions                 |
  +-------------+-------------+
                |
                v
  +---------------------------+
  | Python: execute actions   |  (WinAPI mouse events)
  +-------------+-------------+
                |
                v
  +---------------------------+
  | Python: capture screenshot|  (AFTER actions)
  | - full screen BitBlt      |
  | - crop to working area    |
  | - optional resize         |
  | - encode PNG -> base64    |
  +-------------+-------------+
                |
                v   GET /state (poll)
  +---------------------------+
  | Browser panel (panel.html)|
  | - load raw_b64            |
  | - draw overlays:          |
  |   * orange action heatmap |
  |   * blue bbox heatmap     |
  |   * labels                |
  | - export annotated PNG    |
  +-------------+-------------+
                |
                v   POST /annotated
  +---------------------------+
  | Python: save annotated    |
  | Python: call VLM API      |
  +-------------+-------------+
                |
                v
        next turn begins
```

## Coordinate system (normalized 0..1000)

All model-facing coordinates are normalized integers in [0..1000] relative to the current working area.

- (0,0)       = top-left of working area
- (1000,1000) = bottom-right of working area
- (500,500)   = center of working area

This is the only coordinate system used in:
- VLM JSON actions and bboxes
- Engine in-memory state (S.actions, S.bboxes)
- Panel overlays and labels
- JSONL turn logs

Pixel coordinates exist only at the boundaries:
- When cropping the screen capture (working area selection)
- When physically executing mouse actions (SetCursorPos requires pixels)

Mapping details:
- CAPTURE_CROP defines the working area as a normalized rectangle on the full screen.
- Action coordinates are normalized within that same working area.
- Python converts normalized -> pixel using the current screen size and CAPTURE_CROP.
- The panel converts normalized -> canvas pixels using the decoded screenshot dimensions.

Practical effect:
- Setting CAPTURE_CROP to a quadrant "sandboxes" both what the model sees and where it can click/drag.


## Capture pipeline

The capture pipeline is designed for quality and simplicity (do not change quality/encoding behavior).

Steps:
1) Capture full screen into BGRA bytes using GDI BitBlt (SRCCOPY | CAPTUREBLT).
2) Crop BGRA bytes to the working area rectangle (CAPTURE_CROP) after converting that rect from normalized -> pixels.
3) Optional resize using GDI StretchBlt (HALFTONE) via _stretch_bgra.
4) Encode as PNG and base64.

Resizing controls:
- If CAPTURE_WIDTH and CAPTURE_HEIGHT are both > 0, they fully specify output resolution.
- Otherwise, CAPTURE_SCALE_PERCENT can downscale uniformly after crop.
  - Example: 25 means 25% of the post-crop pixel dimensions.
  - 100 means no scaling.
  - Values <= 0 disable scaling (treated as 100 in practice).

Important:
- Resizing happens AFTER cropping.
- Action execution is not affected by the resized screenshot; actions map to real pixels using the working area mapping.


## Browser panel: overlays and annotation

The panel is a pure UI/annotation stage. It does not read any files from disk.
It polls /state and uses raw_b64 in memory.

Overlay layers:
- Base image: decoded raw PNG from raw_b64
- Heat layer (ctxHeat):
  - Orange heatmap: action locations (click/move/drag endpoints)
  - Optional trail across multiple turns (fade and shrink)
  - Blue bbox heatmap: model-provided bboxes (regions of interest)
- Label layer (ctxLabel):
  - Action index and coordinates text

The annotated screenshot is produced by compositing base + overlays and exporting to PNG, then POSTing to /annotated.

Heatmap trail behavior:
- If UI_CONFIG.executed_heat.trail_turns == 1:
  - Heatmap is one-turn only (no persistence).
- If trail_turns > 1:
  - The panel keeps the last N turns of actions in a small in-memory ring buffer.
  - Each older turn is drawn with lower alpha.
  - Optional shrink (trail_shrink < 1.0) reduces radius and pulls drag endpoints toward their midpoint.

This visual feedback provides "memory" without adding stateful logic to Python.
It can reduce looping when actions cause subtle or no visible pixel change.


## Local HTTP API

The Python engine hosts a local HTTP server (asyncio streams) on HOST:PORT.

GET /
- Serves panel.html.

GET /config
- Returns JSON used by the panel:
  - ui: UI_CONFIG
  - capture_width/capture_height: informational only
- The panel MUST treat the screenshot itself as the source of truth for image dimensions.

GET /state
- Returns the engine state snapshot:
  - phase: engine phase string
  - error: last error (optional)
  - turn: current turn count
  - msg_id: increments per turn
  - pending_seq: expected seq for the next /annotated post (equals current turn)
  - annotated_seq: last annotated seq received
  - raw_b64: base64 PNG of the post-action screenshot (after crop/resize)
  - bboxes: list of bbox dicts (normalized coords)
  - actions: list of action dicts (normalized coords)
  - observation: model-produced observation string
  - vlm_json: last raw VLM JSON text (for UI display/debug)

POST /inject
- Body: {"vlm_text": "<string>"}
- Injects a VLM JSON string into the loop (used for the first turn if BOOT_ENABLED is False or for manual testing).

POST /annotated
- Body: {"seq": <int>, "image_b64": "<base64 png>"}
- The panel submits the annotated screenshot for the current pending_seq.
- The engine validates seq against pending_seq and rejects mismatches.

CORS:
- The server responds to OPTIONS and includes permissive headers in its response writer.
- If you deploy this beyond a trusted local environment, restrict origins and add an auth token.


## VLM API integration

The engine calls a VLM endpoint using an OpenAI-compatible chat completions schema:

- POST API_URL
- JSON payload includes:
  - model, temperature, top_p, max_tokens
  - messages:
    - system: SYSTEM_PROMPT
    - user content:
      - text: observation
      - image_url: data:image/png;base64,<annotated>

The VLM response is expected to be a JSON string matching the schema described in SYSTEM_PROMPT:
{
  "observation": "...",
  "bboxes": [{"x1":..,"y1":..,"x2":..,"y2":..}, ...],
  "actions": [{"name":"click","x1":..,"y1":.., ...}, ...]
}

Parsing behavior:
- The engine attempts json.loads(response_text).
- If parsing fails, it tries to extract the first {...} block.
- On failure, it treats the entire text as observation and returns no actions/bboxes.
- Coordinates are clamped into [0..1000].

Supported action names (case-insensitive, normalized to lowercase):
- move
- click
- right_click
- double_click
- drag (requires x2,y2)

Execution safety:
- PHYSICAL_EXECUTION can be set to False to disable real mouse movement/clicking while still running the loop.


## Disk artifacts and logging

Each run has its own run directory under RUNS_DIR.

Example:
runs/run_0007/

Logging:
- main.log is written to the run directory when LOG_TO_FILE is True.

Artifacts layout toggle:
- LOG_LAYOUT = "flat" (recommended)
  - Flat images in the run directory:
    - turn_0001_raw.png
    - turn_0001_annotated.png
    - ...
  - Append-only JSONL log:
    - turns.jsonl
      - Each line is one JSON object.
      - Two records per turn:
        - stage="raw" includes observation, bboxes, actions, raw_png
        - stage="annotated" includes annotated_png

- LOG_LAYOUT = "turn_dirs" (legacy)
  - Per-turn subfolders:
    - turn_0001/vlm_output.json
    - turn_0001/screenshot_raw.png
    - turn_0001/screenshot_annotated.png
    - ...

JSONL record examples:
{"turn":1,"stage":"raw","observation":"...","bboxes":[],"actions":[...],"raw_png":"turn_0001_raw.png"}
{"turn":1,"stage":"annotated","annotated_png":"turn_0001_annotated.png"}

The panel never reads these files; they are for offline inspection, replay, and debugging.


## Configuration reference (config.py)

Network and UI:
- HOST, PORT
  - Local HTTP server bind address and port.
- LOG_LEVEL, LOG_TO_FILE
  - Logging verbosity and whether to write main.log into the run directory.
- UI_CONFIG
  - Panel overlay settings (see below).

VLM connection:
- API_URL
  - OpenAI-compatible chat completions endpoint.
- MODEL
  - Model string sent in the request payload.
- TEMPERATURE, TOP_P, MAX_TOKENS
  - Sampling parameters.
- VLM_HTTP_TIMEOUT_SECONDS
  - 0 or less means infinite timeout (not recommended for production).

Prompt:
- SYSTEM_PROMPT
  - Must describe:
    - normalized coordinates 0..1000
    - output JSON-only requirement
    - meaning of overlays (orange executed heat, blue bboxes)
    - turn limits (max bboxes/actions)

Capture and working area:
- CAPTURE_CROP (normalized)
  - Working area rectangle on the full screen.
  - Example full screen:
    {"x1":0,"y1":0,"x2":1000,"y2":1000}
  - Example top-left quadrant:
    {"x1":0,"y1":0,"x2":250,"y2":250}
- CAPTURE_WIDTH, CAPTURE_HEIGHT
  - If both > 0, force the post-crop frame resolution.
- CAPTURE_SCALE_PERCENT
  - Used only when CAPTURE_WIDTH/HEIGHT are not set (>0).
  - Uniform scaling applied after crop.
- CAPTURE_DELAY
  - Sleep before capture (seconds), useful for UI settling.

Execution:
- PHYSICAL_EXECUTION
  - False disables actual mouse execution.
- ACTION_DELAY_SECONDS
  - Delay between actions.
- DRAG_DURATION_STEPS
  - Number of intermediate move steps for drag.
- DRAG_STEP_DELAY
  - Delay between drag steps.

Run output:
- RUNS_DIR
  - Base directory for run artifacts.
- LOG_LAYOUT
  - "flat" or "turn_dirs".

Boot injection:
- BOOT_ENABLED
  - If True, the engine starts by injecting BOOT_VLM_OUTPUT as the first model output.
  - If False, it waits for POST /inject.
- BOOT_VLM_OUTPUT
  - JSON string used to bootstrap the loop.

UI_CONFIG details:
- executed_heat:
  - enabled: bool
  - radius_scale: float (relative to max(canvasW,canvasH))
  - stops: list of [pos, rgba-string] radial gradient stops
  - trail_turns: int
    - 1 disables persistence (one-turn heat only)
    - N > 1 enables a fading trail for the last N turns
  - trail_shrink: float
    - 1.0 disables shrink
    - < 1.0 shrinks older heat toward midpoints and reduces radius

- bbox_heat:
  - enabled: bool
  - border: rgba-string
  - border_width: number
  - fill_stops: list of [pos, rgba-string] radial gradient stops


## Multi-turn scenarios

Scenario A: Success path (drawing incrementally)
Turn 1:
- VLM outputs observation + actions to click canvas center and start a stroke.
- Python executes actions.
- Screenshot is captured after the click/stroke.
- Panel overlays orange heat on the stroke region and exports annotated.
- VLM receives annotated screenshot; sees both the stroke pixels and the orange overlay confirming action location.
Turn 2:
- VLM narrates next step and draws next stroke in a different area (guided by overlay and story).
- Repeat.

Why it works:
- The model sees the result of its own actions and an explicit "I acted here" overlay, reinforcing causality.

Scenario B: Loop risk without trail (click that produces no visible change)
Turn t:
- VLM clicks an area that does nothing (no pixel change).
- Screenshot after action looks identical to before.
- Orange heat is present only for this turn.
Turn t+1:
- If the model does not notice the one-turn heat and the pixels are unchanged, it may re-click.

With trail_turns=4:
- The last 4 turns of click heat remain visible with fading.
- Even when pixels do not change, the model sees it has already clicked there recently and can choose a different region.

Scenario C: Annotated seq mismatch
- Engine sets pending_seq = turn.
- Panel mistakenly posts /annotated with the wrong seq (or late post from a previous run).
- Engine replies HTTP 409 and does not advance.
- Engine stays in waiting_annotated phase until a correct annotated image arrives.

Scenario D: VLM returns invalid JSON
- parse_vlm_json fails to parse.
- Engine extracts a {...} block if present; otherwise uses raw text as observation.
- No actions are executed.
- The loop continues (depending on injection or subsequent model response).

Scenario E: VLM endpoint hang (timeout)
- If VLM_HTTP_TIMEOUT_SECONDS <= 0, HTTP timeout is infinite.
- A stalled endpoint can block the call thread.
Recommendation:
- Set a real timeout and handle retries upstream if needed.


## Recreating the system from scratch

To rebuild identically:
1) Implement the Python engine:
   - async main that:
     - creates run dir
     - starts AsyncHTTPServer (asyncio.start_server) serving /, /config, /state, /inject, /annotated
     - runs engine_loop coroutine
   - engine_loop that:
     - waits for next_vlm_json event (boot injection or /inject)
     - increments turn
     - parse_vlm_json -> observation/bboxes/actions (normalized 0..1000, clamped)
     - execute_actions with normalized->pixel mapping inside the working area
     - capture_screenshot: full screen BitBlt -> crop -> optional resize -> png base64
     - save raw artifacts + append JSONL according to LOG_LAYOUT
     - wait for /annotated with matching seq
     - save annotated artifacts + append JSONL
     - call_vlm with observation + annotated image
     - feed returned text back into next_vlm_json
2) Implement the panel:
   - poll /state periodically
   - on phase waiting_annotated:
     - load raw_b64 image into an <img>
     - size canvases to naturalWidth/Height
     - clear layers
     - draw bbox heat (blue) from normalized coords
     - draw executed heat (orange) from normalized coords, with optional trail fade/shrink
     - draw labels
     - export annotated PNG (base64) and POST /annotated with seq
3) Keep config in a single config.py module with the keys listed above.

