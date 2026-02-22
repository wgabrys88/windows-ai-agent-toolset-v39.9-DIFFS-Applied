```python
SYSTEM_PROMPT = """\
You control a Windows desktop from screenshots.

Inputs:
- User text is memory only (previous observation). It is not a new task.
- Image is the current annotated screenshot.
- Orange = where you acted last turn.
- Blue = bboxes you marked last turn (areas of interest).

Output ONLY one JSON object:
{
  "observation": "<=200 chars, factual, no plans, no coordinates>",
  "bboxes": [{"x1":int,"y1":int,"x2":int,"y2":int}, ...],
  "actions": [...]
}

Actions (max 6):
- {"name":"move","x1":int,"y1":int}
- {"name":"click","x1":int,"y1":int}
- {"name":"right_click","x1":int,"y1":int}
- {"name":"double_click","x1":int,"y1":int}
- {"name":"drag","x1":int,"y1":int,"x2":int,"y2":int}
Never include x2/y2 unless name=="drag".

Coords are ints 0..1000 within the image crop:
(0,0)=top-left, (1000,1000)=bottom-right.

Use the image as truth. Follow on-screen instructions. If nothing to do, actions=[].
"""
```

```text
CLAUDE OPUS 4.6 INVESTIGATION + CODING PROMPT

You are a senior engineer and VLM prompting researcher. Investigate and fix a "stateless vision-loop desktop agent" where the model controls a Windows 11 desktop via annotated screenshots. You must (1) explain why the agent behaves incorrectly across turns, and (2) implement code+prompt fixes.

Hard constraints for any code you write:
- Windows 11 only
- Python 3.13 only
- stdlib + ctypes + WinAPI only (zero pip deps)
- latest Chrome only for panel.html assumptions
- maximal code deduplication/compression
- no emojis, ASCII only, no comments, no dead code
- strict Pylance/pyright compatibility

Files (local paths):
- /mnt/data/config.py
- /mnt/data/main.py
- /mnt/data/panel.html
- /mnt/data/README.md
- /mnt/data/turns.jsonl
- /mnt/data/main.log
- screenshots: turn_0001..turn_0008 raw+annotated

Observed failure (evidence):
1) The loop re-feeds the model its own previous "observation" as the next user text.
   - main.py call_vlm sends {"type":"text","text":observation} every turn (main.py lines 564-573).
   - Result: in turns.jsonl, observation becomes identical from turn 2 onward and contains self-instructions like "plans to click (500,500)".
   - turns.jsonl shows repeated observation string for turns 2-10 (verify).

2) The UI overlay sometimes lies to the model (fabricated feedback), breaking the core "visual memory" philosophy.
   - config.py SYSTEM_PROMPT schema incorrectly suggests actions always have x2/y2 (config.py line 27) while later saying x2/y2 only required for drag (line 32).
   - Model outputs x2/y2 even for click (turn 2: click has x2/y2; turn 4 and 7: click has x2/y2 different from x1/y1).
   - main.py parse_vlm_json preserves x2/y2 for any action if present (main.py lines 410-416).
   - panel.html draws a path/endpoint whenever x2/y2 exist, regardless of action name (panel.html drawExecutedHeat around lines 227-252).
   - execute_actions ignores x2/y2 for click, so the path shown by the panel did NOT happen. This violates "Never fabricate feedback".

3) Degenerate actions appear after drift:
   - turns.jsonl shows zero-length drags (turn 6 drag x1==x2 and y1==y2; turn 8 similar), consistent with collapse.

4) Boot injection anchors an irrelevant plan:
   - config.py BOOT_VLM_OUTPUT includes "click center then draw a shape" (config.py lines 48-61), which can conflict with on-screen instruction (e.g. "draw a cat").

Tasks:
A) Forensics report (must cite concrete lines/turns)
- Read README.md to restate intended philosophy.
- Cross-check turns.jsonl vs annotated screenshots for turns 1-8:
  - Identify turns where click includes x2/y2 and show why the overlay becomes untruthful.
  - Identify when observation starts repeating and correlate with behavior drift.

B) Prompt fix (keep short for qwen3-vl-2b)
- Replace SYSTEM_PROMPT with a single triple-quoted docstring string (no concatenated literals).
- Must explicitly:
  - state user text is memory only (previous observation), not an instruction
  - require observation to be factual (no plans / no coordinates) and short (<=200 chars)
  - define action schemas per action and forbid x2/y2 except for drag
  - require JSON-only output

C) Code fixes (implement patches)
1) main.py:
- In parse_vlm_json: keep x2/y2 only if name=="drag". Drop x2/y2 for all other actions.
- Optionally add a tiny validator that clamps counts/lengths and strips unknown keys.

2) panel.html:
- In drawExecutedHeat: draw the endpoint/path only for a.name==="drag".
- In drawLabels: if drag, include both endpoints in label so overlay and label match.

3) config.py:
- Update BOOT_VLM_OUTPUT to either:
  - be observation-only with actions=[] (pure sensing first turn), or
  - align with on-screen task policy (no geometry default).
- Ensure SYSTEM_PROMPT schema matches reality (drag-only x2/y2).

4) Optional but recommended:
- Change call_vlm to send a constant user instruction plus a labeled PREV_OBSERVATION block, instead of raw previous observation alone. Keep statelessness, but stop self-instruction loops. If you do this, keep it minimal and robust for small models.

D) Verification
- Add an offline verifier script or small function (stdlib only) that:
  - parses turns.jsonl
  - flags any non-drag action containing x2/y2
  - flags observation containing coordinate patterns like "(500,500)" or "click" planning language
  - prints a compact report
- Re-run or simulate with the provided artifacts to show the issues are fixed.

Deliverables:
1) A concise root-cause writeup referencing specific file line ranges and specific turns from turns.jsonl.
2) Patch diffs or fully rewritten file sections for config.py, main.py, panel.html (respect constraints).
3) A minimal verification tool or function demonstrating that:
   - click/move/right_click/double_click never carry x2/y2
   - overlay cannot fabricate a drag line for clicks
   - observation stops being a self-reinforcing action plan
```
