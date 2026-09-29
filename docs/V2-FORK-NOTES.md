# netplug-me fork notes (branch `v2`)

This is a fork of `tickernelz/opencode-mem` for our own use. `main` tracks upstream unchanged; our changes live on `v2`. We do not open pull requests upstream.

Upstream `main` already includes a native OpenCode v2 adapter (`src/v2/`, PR #311). The npm release 2.26.0 predates it. On top of that adapter we carry two small fixes, both found and verified on OpenCode 2.0.19.

## 1. Map `session.execution.succeeded` to `session.idle`

- **File:** `src/v2/legacy-client.ts` (`toLegacyEvent`), test in `tests/v2-legacy-client.test.ts`.
- **Problem:** the v1 code triggers auto-capture and user-profile learning from a `session.idle` event. v2 has no such event (2.0.19 emits `session.execution.succeeded`, `session.step.ended`, `session.text.*`, ...), and the adapter forwarded events unchanged. Result: auto-capture never ran.
- **Evidence:** with unmodified `main`, a capture-worthy turn produced no auto-capture log activity. With the fix it logs `Auto-capture memory persisted`.
- **Note:** upstream's adapter test simulates a `session.idle` event, so a different v2 build may emit one. The mapping is harmless in that case.

## 2. Health probe under basic auth

- **File:** `src/services/web-server.ts` (`checkServerAvailable`), test in `tests/web-server-health.test.ts`.
- **Problem:** with `webServerApiToken` set, the owner health check requested `/api/stats` with a Bearer header. When HTTP basic auth is enabled, that request gets 401, so the owner looks dead and each plugin instance takes over the next port (4748, 4749, ...).
- **Fix:** when basic auth is enabled, probe `/api/health`, which is exempt from both the API token and basic auth.
- **Evidence:** unmodified `main` with basic auth on listened on 4747, 4748 and 4749; the fix leaves only 4747.

## Build and run

```
bun install --ignore-scripts && (cd web && bun install --ignore-scripts)
bun run build
bun test tests/web-server-health.test.ts tests/v2-legacy-client.test.ts tests/v2-plugin-adapter.test.ts
```

opencode loads the build through `~/.config/opencode/plugins/opencode-mem.js`, which re-exports `dist/v2/plugin.js`. Restart the opencode service after rebuilding.

See `~/.config/opencode/docs/opencode-mem-setup.md` on the dev machine for the network layout and configuration.
