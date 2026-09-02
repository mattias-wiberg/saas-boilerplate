<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

## User questions

When a skill needs an answer from the user, first check whether an ask-question tool is available (for example, `question` or `AskUserQuestion`). Use it instead of asking in chat when available; otherwise ask in plain text. This applies to live agent-user questions. Questions intended for issue or PR comments and generated scripts remain in those artifacts.
