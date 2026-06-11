# Nuxeo LTS 2023 → LTS 2025 upgrade assistant

A Claude Code skill that upgrades a Nuxeo plugin/marketplace project from **LTS 2023** to
**LTS 2025**. Clone this repo, open it in Claude Code, run the command, point it at your
Nuxeo project, and it walks the migration: version bumps, the `javax`→`jakarta` move,
removed/relocated APIs, dropped libraries, test-infrastructure changes, then builds with
JDK 21 and fixes whatever the compiler/tests surface.

## Requirements (on the machine running the upgrade)

- **JDK 21** (Nuxeo 2025 enforces `[21,)`). Verify: `/usr/libexec/java_home -v21` (macOS) or `java -version`.
- **Maven 3.9.x**.
- Access to the **Nuxeo 2025 private artifacts** — LTS releases are subscription-only
  (`packages.nuxeo.com`/your corporate proxy). The build won't resolve `2025.x` without it.
- A 2025-compatible build of your **Studio project** (if you use one).

## Usage

```bash
git clone <this-repo>
cd nuxeo-lts-2025-upgrade
claude            # open Claude Code in this directory
```

Then in Claude Code:

```
/nuxeo-2025-upgrade
```

It will ask for the absolute path to your Nuxeo project and then perform the upgrade
**in that project** (this repo is just the toolbox — it edits your code, not itself).

## What it does

1. Locates and assesses your project (Maven modules, current 2023 versions).
2. Bumps versions: `nuxeo-parent`, platform version, Java 21, package target platform.
3. Applies code migrations (Jakarta EE, dropped libs, removed/relocated APIs) using
   [`reference/migrations.md`](.claude/skills/nuxeo-2025-upgrade/reference/migrations.md).
4. Fixes test-infrastructure changes (search feature, mail server, bulk scrollers, directories).
5. Builds with JDK 21 and iterates on errors until it compiles and tests pass.
6. Reports what changed and flags anything needing your input (e.g. Studio version).

## Scope

- **Server-side Java / Maven** plugin and marketplace-package projects — versions, Jakarta EE,
  dropped/removed/relocated APIs, test-infra, build-and-fix on JDK 21.
- **Studio Web UI** (Polymer) projects — the skill **verifies** 2025 readiness rather than
  rewriting: the Web UI 2023→2025 upgrade is transparent, and the legacy Polymer elements are meant
  to stay as-is. It updates `application.xml`, migrates functional tests if present, and flags real
  breakages. See [`reference/web-ui.md`](.claude/skills/nuxeo-2025-upgrade/reference/web-ui.md).

## Limitations

- It does **not** rewrite legacy Polymer/`<dom-module>` elements (by design — Nuxeo says keep them),
  and it does not migrate the Studio *model* itself (doctypes/workflows) — that's done in Studio.
- The exact `2025.x` patch version and your Studio GAV are environment-specific; the skill asks.
- See the "Known sharp edges" section of the skill for cases that may need manual attention.
