This is a Forge app for Atlassian Confluence Cloud.

Before writing any Forge code, read:
- .claude/forge-cheatsheet.md (gotchas and known-good patterns)
- .claude/manifest-reference.yml (canonical manifest shape)
- .claude/llms.txt (index of canonical docs to WebFetch if you need more)

Runtime: nodejs22.x. UI Kit (React-based), not Custom UI, unless explicitly asked.
Never invent Forge APIs — if unsure, WebFetch the relevant llms.txt entry.
Always show me the manifest and a summary of files before I run `forge deploy`.
