# 2022x Fixes: Windows Portability

The `dev/2022x-bridge` branch's fixes to run this bridge against Cameo Systems
Modeler 2022x were validated on a macOS install (`/Applications/Cameo Systems
Modeler`, 2022x Refresh 2). This note records why those fixes are expected to
carry over to a Windows 2022x install unchanged, and what still needs to be
verified there directly.

## Why the fixes are platform-independent

- **`plugin/build.gradle`** (`sourceCompatibility`/`targetCompatibility`/
  `options.release` set to `11`) — a Java bytecode target, not an OS setting.
  Cameo 2022x (base through Refresh 2) ships an embedded Java 11 JRE; this
  applies identically on Windows.
- **`plugin/plugin.xml`** (dropped `version="2024x"` from the two
  `required-plugin` entries for `com.nomagic.magicdraw.diagramtable` and
  `com.nomagic.magicdraw.visualization.relationshipmap`) — plain XML
  metadata. Dropping the version attribute (matching how No Magic's own
  plugin.xml files declare optional dependencies) means this no longer
  assumes any specific refresh string at all, on any OS.
- **`MatrixHandler.java`, `DiagramHandler.java`, `GenericTableHandler.java`**
  — pure Java syntax rewrites (a Java 15+ text block rewritten as string
  concatenation; Java 16+ pattern-matching `instanceof` rewritten as classic
  cast form) to compile cleanly at `--release 11`. No filesystem or OS calls
  involved.
- **`mcp-server/pyproject.toml`** (`mcp>=1.0.0,<2` pin) — a Python dependency
  constraint, identical on any OS.

Cameo's `lib/*.jar` and `plugins/*/*.jar` files are plain Java bytecode,
identical across Windows/macOS/Linux for the same product version and
refresh. That's why a clean compile against a local macOS install is
meaningful evidence for a Windows install of the same version: the compiled
class files don't depend on which OS produced or will run them.

## What still needs to be verified on Windows

1. Pull this branch and run the existing documented Windows flow (see
   `docs/development/build-and-live-validation.md`):
   ```powershell
   gradlew.bat compileJava -PcameoHome=D:/DevTools/CatiaMagic -Pjdk17Home=D:/DevTools/jdk17/jdk-17.0.18+8
   ```
   against the real Windows 2022x install.
2. Confirm that install's `diagramtable`/`relationshipmap` plugin.xml
   versions (doesn't need to match `"2022x Refresh2"` specifically anymore,
   since the version pin was dropped — just worth knowing which refresh is
   installed).
3. Deploy (`gradlew.bat deploy ...`) and restart Cameo there the same way
   validated on macOS.

## Pre-existing wrinkle (not introduced by this branch's fixes)

`build.gradle`'s `Test` task hardcodes `${jdk17Home}/bin/java.exe` when the
`jdk17Home` property is set — Windows-only path syntax. Harmless if that
property isn't passed, but will fail with a "file not found" if you run
`gradlew.bat test -Pjdk17Home=...` on Windows and that exact path/executable
doesn't exist there.
