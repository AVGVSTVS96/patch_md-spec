---
title: PATCH.md
description: A file format for defining source patches that AI agents can apply and maintain.
version: 0.1.0
lastUpdated: 2026-06-25
---

# PATCH.md

> [!IMPORTANT]
> **Status: early draft (v0.1.0).** This format is new and intentionally small.
> Refinements, proposals, and discussions are welcome.

`PATCH.md` is a file format for defining a source patch that agents can apply
and self-update over time. A patch file holds two things: its **intent**,
written in plain prose, and its **edits**, anchored to the content they change
rather than to line numbers.

Anchoring to content is what lets a patch survive changes to the file it targets.
When an edit stops matching, an agent can read the intent and the old edit, find
the equivalent place in the updated file, and rewrite the edit to fit. The format
describes the patch. The tools built around it do the applying and the repairing.

## File

One patch is one `PATCH.md` file, kept by convention at `<patch-id>/PATCH.md`.
<patch-id> can be a custom or generated name, id, etc.

## Structure

````md
---
id: <patch-id>
summary: <short summary of the patch's intent>
version: <semver>
lastUpdated: <yyyy-mm-dd>
---

# <Patch Title>

## Intent

<What the patch does and why, in plain prose. This is what an agent reads
to re-anchor an edit when the target file changes.>

```diff file=<target/path>
@@ optional hint @@
 context
+added line
```
````

The `## Intent` section is the conventional place for the patch's purpose and
re-anchoring guidance. It is the only heading convention defined by this spec.

A patch can list several targets by including several edit fences. The target file
is declared on each edit fence with `file=<target/path>`. Any headings used to
organize edits are optional prose, not part of the patch format.

## Edit forms

An edit is a fenced code block that defines the patch's current code.
To start, we have added two recommended formats, use whichever you prefer:

**Search/replace block form** shows the full before and after, anchored on the whole replaced region:

```patch file=dist/package-manager-cli.js
<<<<<<< SEARCH
                    console.log(chalk.green(`Updated ${APP_NAME}`));
=======
                    console.log(chalk.green(`Updated ${APP_NAME}`));
                    try { (await import("node:child_process")).spawnSync("pi-patcher", ["reconcile"], { stdio: "inherit" }); } catch {}
>>>>>>> REPLACE
```

**Hunk form** is diff style and marks only the changed lines:

```diff file=dist/package-manager-cli.js
@@ pi self_update success branch @@
                     console.log(chalk.green(`Updated ${APP_NAME}`));
+                    try { (await import("node:child_process")).spawnSync("pi-patcher", ["reconcile"], { stdio: "inherit" }); } catch {}
```

- `-` marks a removed line and `+` marks an added line.
- The fence info string must include `file=<target/path>` so tools can apply the edit without interpreting prose. The value may be bare (`file=dist/foo.js`) or quoted (`file="path with spaces/foo.js"`). Fences without `file=` are examples/prose, not mechanical edits.
- Optional: For additional context, or agent hints:
  - `@@ ... @@` header as a hint or more context.
  - `> note:` line directly before an edit for more context or a hint.

## Applying edits

Every edit has a before and an after. The before must appear exactly once in the
target file, so there is never any doubt about where the edit goes. Applying
replaces the before with the after. In hunk form the before is the context plus
the removed lines, and the after is the context plus the added lines.

Because the match is unique, applying the same patch twice is safe, and a tool
can read each edit's state directly:

- **applied**: the after is present
- **pending**: the before is present
- **drift**: neither is present

All edits in a patch apply together or not at all. If any edit fails, a tool
restores the file to how it started.

## Splitting a patch

A patch can touch several files, but when one grows large or bundles unrelated
changes, split it into smaller sub-patches. Smaller patches are easier to
maintain, and easier for an agent to heal.

Sub-patches should be nested inside their parents' directory, and the parent/child
relationships are defined in frontmatter. A parent patch lists its parts, and each
sub-patch names its parent:

```
bootstrap-hook/
├── PATCH.md
├── hook-insert/
│   └── PATCH.md
└── hook-config/
    └── PATCH.md
```

```
# parent patch frontmatter
parts: [hook-insert, hook-config]

# sub-patch frontmatter
parent: bootstrap-hook
```

## Agent Driven Healing

When an edit no longer matches its target, the patch has drifted. Based on the `PATCH.md`,
an agent can automatically fix and update the patch when the target changes.

Snapshot the target before editing and restore it if anything fails. A tool can also
let the agent decline, for example when the feature a patch depended on has been removed,
instead of forcing a change that no longer fits.

Tool developers and/or patch authors can decide how complex of a change their agent
is allowed to heal automatically.

## Example

The `bootstrap-hook` patch from [pi-patcher](https://github.com/AVGVSTVS96/pi-patcher), written as a single PATCH.md:

````md
---
id: bootstrap-hook
summary: Re-run `pi-patcher reconcile` after pi finishes updating itself.
version: 0.1.0
lastUpdated: 2026-06-25
---

# Bootstrap hook

## Intent

Patches Pi to run `pi-patcher reconcile` after pi finishes updating itself.

After pi's `pi update` finishes updating pi, run `pi-patcher reconcile` so every
installed patch is re-applied to the freshly updated install.

The hook goes right after the existing `Updated ${APP_NAME}` success log in pi's
self-update path. It must stay valid in pi's compiled ESM output, and it must not
change update behavior if `pi-patcher` is missing or fails to start.

> note: end of the self-update branch, right after `Updated ${APP_NAME}`

```diff file=dist/package-manager-cli.js
@@ pi self update success branch @@
                     console.log(chalk.green(`Updated ${APP_NAME}`));
+                    try { (await import("node:child_process")).spawnSync("pi-patcher", ["reconcile"], { stdio: "inherit" }); } catch {}
```
````
