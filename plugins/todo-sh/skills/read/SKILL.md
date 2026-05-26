---
name: read
description: Read the user's personal to-do list by running `todo.sh ls` and returning its output verbatim. This is a read-only context-provider skill — it MUST NOT add, complete, edit, prioritize, archive, or otherwise mutate any to-do item. Trigger on /todo-sh:read, when the user asks to see / show / list / dump their to-do list or todo.txt, or when another skill needs the current to-do list as context. Do NOT trigger on the harness Task tools (TaskCreate / TaskUpdate / etc. — those are unrelated in-conversation task tracking), and do NOT trigger on writing TODO comments in source code.
---

# read

Surface the current to-do list as plain text so the caller (another skill or the user) can decide what to do with it.

Run the following and return the output verbatim:

```bash
todo.sh ls
```

**Read-only guardrails — never run any of these:**

- `todo.sh add` / `todo.sh a`
- `todo.sh do`
- `todo.sh rm` / `todo.sh del`
- `todo.sh pri` / `todo.sh p`
- `todo.sh archive`
- Any direct edits to `~/todo.txt` or `~/.todo/todo.txt`

The output is plain text; leave interpretation and action to whoever invoked this skill.
