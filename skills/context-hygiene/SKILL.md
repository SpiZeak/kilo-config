---
name: context-hygiene
description: Use when handling large tool outputs, quoting logs or command failures, long sessions, or context-window pressure — including big-output piping, tail-before-quoting, truncated postmortems, and compaction advice.
---

# Context Hygiene

Rules for keeping the context window lean and the conversation readable.
Load this skill when outputs are large, a session runs long, or context
pressure is a risk.

## 1. Huge output

When a `bash` command, `read`, or `grep` would return more than ~2k lines
or 50KB, do not let it flood context. Pipe to a file under `/tmp/kilo/`
then `Read` the relevant slice:

```
cmd > /tmp/kilo/out.txt 2>&1 && wc -l /tmp/kilo/out.txt
```

## 2. Keep command outputs concise

Summarize large files or logs instead of reading them in full if only a
section is needed.

## 3. Tail before quoting

Use `tail -n 50` not `cat` when debugging build or test failures. Errors
are almost always at the end.

## 4. Truncate postmortems

A stack trace only needs the failed frame plus the nearest user-owned
frame above it. Drop framework internals.

## 5. Replace entire files carefully

When `read` returns 100% of a large file, re-quoting it back is wasteful.
Reference ranges with `file:start-end` and `read` the new range after
edits.

## 6. Keep supplementary rules out of this file

Use the `instructions` glob in `kilo.json` to pull in supplementary rules
(lint configs, style guides, ADRs) without bloating instruction files.

## 7. Long sessions

If a session runs long, suggest `/compact` before context window pressure
degrades output quality.

## 8. Tool errors

Do not echo back small tool errors verbatim; state what failed and the
one next action the user can take.
