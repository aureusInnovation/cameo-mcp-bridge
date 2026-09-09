# Preflight

- Probe the bridge before assuming it is live:
  - `cameo_probe_bridge`
  - `cameo_status`
  - `cameo_get_capabilities`
  - `cameo_get_project`
- A healthy bridge is not enough; `cameo_get_project` must show an open project or model mutation will fail.
- When Java bridge code changes, rebuild, redeploy, and fully restart Cameo before trusting live results.
- Gradle itself needs a Java 17+ JDK to run. The plugin's *compiled bytecode target* varies by branch — check `plugin/build.gradle`'s `sourceCompatibility`/`options.release` before assuming Java 17 output (`master` targets 17 for Cameo 2024x; `dev/2022x-bridge` targets 11 to match Cameo 2022x's embedded JRE).
- On a Windows dev machine with the repo mounted over a UNC path (e.g. `Z:\cameo-mcp-bridge`), Git or shell behavior may need UNC-safe handling. Not applicable on a local checkout (e.g. macOS).
