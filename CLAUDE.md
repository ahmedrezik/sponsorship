# Sponsorship GAP Analysis — Claude Code Instructions

Before touching any file, read FEATURE.md. It is the source of truth.
Do not make decisions that belong in the feature description. If something is unclear, ask.

## Commands

```bash
# Setup (once)
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Run
python main.py

# Logs (live, pretty)
tail -f logs/app.log | python -m json.tool
```

## What to do when implementing

1. Read the relevant FEATURE.md section first
2. Implement exactly what it says — no additions
3. If the spec is ambiguous, stop and ask before guessing

## Hard rules

- Prompts live in `prompts/` as `.txt` files. Never inline them in Python.
- `$var` placeholders only (string.Template). Never `str.format()` on prompt strings.
- Log event names match the spec exactly: `"step1.start"`, `"pipeline.complete"`, etc.
- If a file isn't in FEATURE.md §2, don't create it.
- Don't add error handling for cases the spec doesn't mention.

## Gotchas

- Azure client uses `timeout=None` intentionally — Step 1 can take 10+ min.
- Port 8000 conflicts if a previous instance is still running: `lsof -ti :8000 | xargs kill`
- `AsyncAnthropicFoundry` is the correct class, not `AsyncAnthropic`.
- `logs/` is gitignored but auto-created by `_configure_logging()` at startup.
