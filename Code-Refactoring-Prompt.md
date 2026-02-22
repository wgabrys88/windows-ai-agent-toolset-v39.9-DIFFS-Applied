## HTML -> BASE64, temp = 1, Claude Opus 4.6

## This is useful to send html as base64 to not break the chat with AI
```
import base64
with open("panel.html", "rb") as f:
    print(base64.b64encode(f.read()).decode("ascii"))
```

## This is the prompt itself:
```
You are an elite systems programmer who produces extremely clean, minimal, high-performance, and reliable code under hard constraints.

PROJECT: Franz — Full Vision-Action Loop Agent (major refactor)

TASK: Completely refactor and modernize the Franz system.

STRICT REQUIREMENTS — follow every rule without exception:

• Python 3.13 only. Aggressively use every modern 3.13 feature that reduces code or improves clarity (match-case with guards, walrus operator, structural pattern matching, type statements, etc.).
• Windows 11 only — delete all legacy, compatibility, or cross-platform code.
• Latest Google Chrome only for panel.html.
• Zero pip dependencies ever. Only stdlib + ctypes + WinAPI.
• Maximum code deduplication and compression.
• No emojis, no non-ASCII characters, no comments, no dead/commented code.
• Strict Pylance/pyright compatibility and perfect IntelliSense.
• Keep and optimize the pure-Python PNG encoder (make it compact).
• VLM must output strict JSON only — update the system prompt and all parsing logic accordingly (remove all regex/PART 1/PART 2 logic).
• Use a modern asyncio-based server and engine coordination instead of threading + ThreadingMixIn to eliminate threading problems. Prefer asyncio.Event, asyncio.Lock, asyncio.Task, and stdlib asyncio HTTP handling where possible while keeping zero dependencies.
• Ensure the entire system has no timeouts: long-running connections, keep-alive, stable polling, and ability to run autonomously for 10+ hours with near-zero memory growth and no resource leaks.
• SINGLE SOURCE OF TRUTH RULE: The exact image the user sees in the browser panel (raw screenshot + all overlays/heatmaps/labels rendered on the canvas) must be the precise image that gets exported to base64 PNG and sent back to the engine via /annotated. The VLM must receive exactly what the user visually sees — no discrepancy allowed.


Python main will be provided in next message, await for it then await once more, for the config and html file which will be encoded in base64.
