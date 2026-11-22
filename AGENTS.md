# fork-omp-headroom

Fork of [DarkPhilosophy/omp-headroom](https://github.com/DarkPhilosophy/omp-headroom) v0.1.4 — the omp plugin that compresses provider payloads through a local Headroom proxy and renders the terminal HUD. Upstream = `git+https://github.com/DarkPhilosophy/omp-headroom.git`; this fork = `git@github.com:woss/fork-omp-headroom.git` (GitButler workspace, base e7788d0 on 56d3ffb).

This fork exists to carry fixes the upstream doesn't have yet. Work here, PR back upstream where the fix is generic.

## The consumer you must not break

`~/projects/woss/paseo-headroom` is a Paseo plugin (sibling repo) that polls this proxy's `/stats` and renders a panel/pill/popover. Its contract (`shared/headroom.ts` there) reads these `/stats` fields — additive changes only, never rename/remove:

- `summary`: `api_requests`, `primary_model`, `compression.{requests_compressed, avg_compression_pct, best_compression_pct, best_detail, total_tokens_removed, total_tokens_before}`, `uncompressed_requests.{prefix_frozen, no_compressible_content}`, `tip`
- `savings.per_project` keyed by omp session UUID: `requests`, `tokens_saved`, `total_input_tokens`, `savings_percent`, `last_activity_at`
- `agent_usage.agents[].models` (model → request count)

Run the Paseo plugin's typecheck against your `/stats` changes: `cd ~/projects/woss/paseo-headroom && npm run typecheck`.

## Known bugs to fix

### 1. Pricing: unknown-model fallback is $3/1M flat

Every `cost_usd` the plugin/proxy writes is computed by `pricing/litellm_pricing.py` → `resolve_litellm_model(model)` with candidates `("glm-5.3-flash", "openai/glm-5.3-flash", "anthropic/…", "google/…", "mistral/…", "deepseek/…", "minimax/…")`. LiteLLM's `model_prices_and_context_window.json` has **no bare `glm-5.3-flash` key** — the correct entry exists only as `zai/glm-5.3-flash` ($1.5e-07/token input = $0.15/1M, the real opencode-zen rate per models.dev). All candidates miss → `estimate_cost` falls back to a flat $3/1M (measured: $3.0005/1M on session 01a2006e) — 13–75× over the real opencode-zen rate for flash-tier models.

Fix: at event time the plugin knows the omp provider it's wrapping — pass the provider identity into the resolution (or try models.dev, which lists opencode-zen/glm-5.3-flash correctly) before the flat fallback. Also consider making the fallback price explicit/configurable instead of a silent constant.

Every historical `cost_usd` in `~/.headroom/savings_events.jsonl` is baked at the wrong rate; the proxy sums them as-is and never recomputes. A migration that re-derives `cost_usd = saved × correct_rate` per event (all events are glm-5.3-flash today) is in scope.

### 2. Session attribution: expose the omp session id on proxied requests

The plugin knows `state.sessionId` per request. Downstream consumers (Paseo agents) need the reverse binding: which omp session owns which request/agent. Fix: stamp proxied requests with the omp session id (header or `/stats` event field), and/or expose `GET /session-map` returning `{omp_session_id → last_activity, provider}`. This kills the 20-minute last-activity proximity heuristic the Paseo panel currently uses to guess agent↔session binding.

### 3. Sessions with zero savings are invisible

Sessions whose requests are all prefix-frozen (or produce no strict reduction) get no `savings_events` row, so per-session views can't see or attribute them at all (real case: session 01a20022, 4.8 MB transcript, zero events). Consider recording zero-savings requests as events with `saved: 0`, or a separate `seen_sessions` structure in `/stats`.

## Local environment (this machine)

- Persistent proxy: launchd `com.headroom.default` → `~/.headroom/deploy/default/run-headroom.sh` → `~/.local/bin/headroom install agent run`. Currently 0.44.0 (the omp venv at `~/.omp/agent/headroom-venv` was updated to 0.44.0 on 2026-11-22; older 0.34.0 proxy processes may linger after restarts — check `lsof -iTCP:8787`).
- LiteLLM pricing maps patched with a bare `glm-5.3-flash` entry ($1.5e-07/token, matching `zai/glm-5.3-flash`) in BOTH: `~/.omp/agent/headroom-venv/lib/python3.13/site-packages/litellm/model_prices_and_context_window_backup.json` and `~/.local/share/uv/tools/headroom-ai/lib/python3.13/site-packages/litellm/model_prices_and_context_window_backup.json`. **Headroom updates replace site-packages and wipe these patches** — re-apply after every update until the fork/upstream fixes resolution.
- Durable ledger: `~/.headroom/savings_events.jsonl` — per-request events with `ts, before, after, saved, cost_usd, model, client, source, pid`. `cost_usd` is baked at event time at the wrong rate for pre-fork events.
- Proxy restarts reset proxy-wide counters; the durable ledger persists. Restart cleanly with `headroom install restart` (a failed bootstrap may need `launchctl bootout gui/501/com.headroom.default` + `headroom install start`).

## Verification

- `bun test` (tests/ exist), `biome` lint per `biome.json`.
- Typecheck: `bunx tsc --noEmit` per `tsconfig.json`.
- Live: with the proxy running, `curl -s localhost:8787/stats | python3 -m json.tool` — confirm the fields above still resolve, and that a fresh glm-5.3-flash session appears in `savings.per_project` within one request of traffic.
- The Paseo consumer: `~/projects/woss/paseo-headroom` must typecheck clean (`npm run typecheck`) against any `/stats` shape change.

## Constraints

- Everything stays local; no telemetry.
- Don't break omp compatibility (this plugin loads inside omp's process — `before_provider_request` hook, OpenAI/Anthropic/Responses wire formats).
- The omp session UUID is the attribution key — keep it stable in `/stats` output.
