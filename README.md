# Forge + Claude context pack

Three files that make Claude Code actually useful for Forge work instead of confidently wrong.

If you were at the talk, this is the pack you watched. If you weren't: Claude's training barely covers Forge, so it hallucinates manifest fields, invents imports, and uses deprecated UI Kit syntax. This pack fixes that in about 15 minutes of setup.

## What's in here

```
.claude/
  forge-cheatsheet.md       gotchas, patterns, common errors
  manifest-reference.yml    known-good annotated manifest
  llms.txt                  URLs Claude fetches on demand
CLAUDE.md                   project-level rules, auto-loaded by Claude Code
```

That's it. No tooling, no frameworks, ~300 lines of markdown.

## Drop it into your project

```bash
cd your-forge-project
cp -r /path/to/pack/.claude .
cp /path/to/pack/CLAUDE.md .
```

Start Claude Code in that folder. It picks up `CLAUDE.md` automatically on the first prompt. You should see it read the cheatsheet before writing anything.

## Try it

Ask Claude to add something small to your app and watch what changes. A good first test:

> Add a resolver that returns the current user's display name, and wire it into the existing macro. Walk me through the manifest and the handler before I deploy.

If the pack is working, Claude will read the cheatsheet first, use `api.asUser().requestConfluence(route\`...\`)` without being told, and pause before suggesting `forge deploy`.

## Make it yours

The pack ships configured for Confluence macros, React-based UI Kit, nodejs22.x. If your work is different, edit the files:

- **Jira apps** — swap the API examples in `forge-cheatsheet.md`, point `llms.txt` at the Jira REST docs instead
- **Legacy UI Kit (`@forge/ui`)** — rewrite the "Two UI patterns" section to lead with the legacy pattern since that's what you're on
- **Custom UI** — replace the manifest reference and the UI Kit section entirely
- **Your team's conventions** — add them to `CLAUDE.md` ("we always use TypeScript", "every resolver returns `{ok, data, error}`", whatever)

The pack is a template, not a product. Edit freely.

## Keep it alive

The pack earns its keep by capturing every gotcha so you only hit each one once. When Claude generates something broken:

1. Figure out the real fix (deploy error, docs, trial and error).
2. Add a row to the "Common errors" table in `forge-cheatsheet.md` or append the gotcha to the right section.
3. Commit it with the rest of your code.

Six months from now, future-you or a teammate hits the same error. The pack already has the answer. That's the whole game.

I added two rows to this version of the cheatsheet while preparing the talk — one for a missing `render: native` and one for forgetting `npm install` before `forge deploy`. Both bit me live. Both are in there now.

## Example app

If you want to see the pack driving a real build, the `demo-app-reference/` folder has a working Confluence macro that lists page attachments:

```bash
cd demo-app-reference
npm install
forge register                                                   # creates app, writes ID into manifest.yml
forge deploy
forge install --product confluence --site your-site.atlassian.net
```

Then add the "Page Attachments" macro to any Confluence page that has attachments.

## The bigger idea

This isn't a Forge trick. The same three files work for any framework that's thin on general LLM training data — proprietary SDKs, internal platforms, niche DSLs, your company's weird integration layer. Copy the shape, change the contents, keep it updated.

If you build a version for another framework, I'd love to see it.

---

Built for the "The 3 files that turn Claude into a Forge expert" talk — Hernan Montes, Movonte. Use it, fork it, change it.
