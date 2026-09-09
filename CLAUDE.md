# cameo-mcp-bridge — dev/2022x-bridge branch

This branch exists to keep the Cameo MCP Bridge working against **CATIA Magic /
Cameo Systems Modeler 2022x** (validated on 2022x Refresh 2, locally installed
at `/Applications/Cameo Systems Modeler`). `master` targets 2024x. Read
`AGENTS.md` and `README.md` first for the general architecture and contract —
this file only covers what's specific to keeping *this branch* 2022x-compatible.

## Hard constraints — do not regress these

The bridge failed on 2022x before this branch existed because it silently
assumed 2024x. Every constraint below exists to prevent that regression:

- **Java bytecode target stays at 11** (`plugin/build.gradle`:
  `sourceCompatibility`/`targetCompatibility`/`options.release`). Cameo 2022x
  (base through Refresh 2) ships an embedded Java 11 JRE; classes built above
  release 11 fail to load with `UnsupportedClassVersionError`. Don't bump this
  back to 17 without re-verifying the target install's actual embedded JVM
  version (Help > About, or the JRE bundled next to the install) — "2022x
  Refresh 2" alone is not enough evidence; ours reports JVM 11.0.23.
- **No Java 12+ language syntax** anywhere in `plugin/src/`: no text blocks
  (`"""`), no pattern-matching `instanceof`/`switch`, no records, no sealed
  classes. Before committing, check:
  ```bash
  grep -rn '"""' plugin/src
  grep -rnE "instanceof [A-Za-z_$][A-Za-z0-9_$.<>]* [a-zA-Z_$][A-Za-z0-9_$]*\b" plugin/src
  ```
- **`plugin/plugin.xml`'s `required-plugin` entries stay version-attribute-free**
  for `com.nomagic.magicdraw.diagramtable` and
  `com.nomagic.magicdraw.visualization.relationshipmap`. Don't hardcode a
  version string (e.g. `"2024x"`, `"2022x Refresh2"`) — No Magic's own
  plugin.xml files omit this attribute for optional deps, and omitting it is
  what makes the plugin load regardless of which 2022x refresh is installed.
- **`mcp-server/pyproject.toml` keeps `mcp>=1.0.0,<2`** until `cameo_mcp/server.py`
  is deliberately migrated off `mcp.server.fastmcp.FastMCP` to the v2
  `MCPServer` API. An unpinned `pip install -e .` today grabs `mcp` 2.x and
  breaks the import outright — this isn't 2022x-specific, it'll bite the
  2024x/master install too if its venv is ever rebuilt from scratch.

## Validating a change on this branch

A clean `master`-style compile does **not** prove 2022x compatibility — only a
build against real 2022x jars does. Cameo's `lib/*.jar` and `plugins/*/*.jar`
are plain, OS-independent Java bytecode, so this local macOS install is valid
evidence for a Windows 2022x install too (see
`docs/development/2022x-windows-portability.md`).

```bash
cd plugin
JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home \
  sh gradlew compileJava test -PcameoHome="/Applications/Cameo Systems Modeler"
```

(`gradlew` isn't marked executable in this checkout — use `sh gradlew`, or
`chmod +x` it, rather than reintroducing a Windows-only invocation.)

Per `AGENTS.md`: do not deploy the plugin or restart Cameo as a side effect of
routine validation — `compileJava`/`test`/`assemblePlugin` are sufficient to
verify a change. Deploying (`gradlew deploy`) and restarting Cameo is a
deliberate, explicitly-requested step, since it mutates the user's running
install. After deploying, confirm `/api/v1/status` and `/api/v1/capabilities`
report the rebuilt plugin version before trusting any new endpoint.

## Known runtime gap (compile-clean, not behavior-clean)

`cameo_get_active_diagram` on this 2022x install returns a `warnings` array:
`com.nomagic.magicdraw.uml.symbols.DiagramPresentationElement.getSelectedPresentationElements unavailable`
and the `getSelectedElements` counterpart. These methods exist in 2024x but
not in 2022x's OpenAPI — compiling clean only proves the *symbols* this
codebase references resolve; it says nothing about whether every handler
*behaves* the same across versions. Treat any `"...unavailable"` warning
surfaced through a handler's reflection/probe path as a real compatibility
gap to track, not noise — and prefer that same probe-and-degrade pattern
(see `OptionalCapabilitySupport`) over a hard compile-time dependency when
adding anything that might not exist on 2022x.

## If this branch is ever reconciled with master

`master`'s Java 17 target and hardcoded 2024x plugin-version pins are exactly
what broke 2022x. If cherry-picking from master or merging it in, re-check
the two constraints above — a merge can silently reintroduce them.
