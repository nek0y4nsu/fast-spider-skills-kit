---
name: fast-spider
description: >-
  Browser environment patching toolkit for crawler reverse engineering.
  v2.1 prototype engine with config-driven architecture, 5-layer defense,
  seed-deterministic fingerprinting, and document.all access monitoring.
  Keywords: 补环境, 爬虫逆向, 签名, JSVMP, vm sandbox, fake_env,
  trace_env, capture_sdk, persistent_signer, Toolkit, fingerprint, 加签.
---

## 1. Core Architectural Skills

### 1.1 Config-Driven Architecture

Migrated from hardcoded `fake_env.js` templates to declarative JSON configuration.
One config file per target — no code changes between platform deployments.

```
Before:  copy sign_template.js → hand-edit SDK paths → debug crash → repeat
After:   cp config/templates/default-anti-detect.json config/targets/<platform>.json
         → edit 5 fields → Toolkit.loadConfig('./config/targets/<platform>.json')
```

**Entry point:** `src/index.js`
```js
const Toolkit = require('./src');
const config  = Toolkit.loadConfig('./config/targets/<platform>.json');
const win     = Toolkit.createBrowserFromConfig(config, {
    target: { test_url: '<target-test-url>' }
});
```

**Key advantage:** SDK rotation? Copy the old config, point `sdk_path` to the new
bundle, run Phase 7 validation. Core logic untouched.

### 1.2 Five-Layer Defense System

Each layer fails independently — an attacker must breach all five to detect
the sandbox. This is the toolkit's core competitive moat.

| Layer | Mechanism | File | Breach Consequence |
|-------|-----------|------|--------------------|
| **L1: Prototype Chain** | `instanceof` + `Symbol.toStringTag` on 8 constructors (Window, Navigator, PluginArray, Plugin, MimeType, MimeTypeArray, EventTarget, WindowProperties) | `src/env/core.js` | Detection degrades to `[object Object]` — triggers monitoring alert |
| **L2: Native Marking** | `Function.prototype.toString` hijack via Symbol-based `safefunction()`. All stubbed methods return `function name() { [native code] }` | `src/env/utils.js` | Function identified as non-native — logged with stack trace |
| **L3: Property Locking** | `PrototypeBuilder.lockAll()` + `lockPrototypes()` seals all 8 prototypes. SDK `Object.defineProperty` attempts → `TypeError` | `src/env/prototype.js`, `src/env/core.js` | Tamper attempt crashes SDK — visible in stderr |
| **L4: Access Monitoring** | `document.all.getAccessLog()` records every property access with `{prop, time, stack}`. `debugAll: true` enables globally | `src/env/document-all.js` | Even if bypassed, monitoring captures full call chain of detection attempt |
| **L5: Config Isolation** | Each target gets its own JSON config. GPU profile, seed, plugins — all per-target | `config/targets/*.json` | One target's leak doesn't compromise others |

```
SDK probes navigator.plugins
  → L1 passes (instanceof PluginArray ✓)
  → L2 passes (item.toString() → [native code] ✓)
  → L3 active (cannot override PluginArray.prototype.item)
  → L4 records: {prop:"plugins", time:..., stack:"at SDK_init:42"}
  → L5 isolates this target's plugin config from other platform configs
```

### 1.3 Observability Design

Transforms "guess-based debugging" into "transparent diagnostics."

**document.all access monitoring:**
```js
const win = Toolkit.createBrowser({ debugAll: true });
// ... run SDK in vm ...
const log = win.document.all.getAccessLog();
// → [{prop: "length", time: 1779091713398, stack: "at SDK._$clt (signer.js:1420)"}]
```

**beforeParse lifecycle hook**:
```js
const win = Toolkit.createBrowser({
    beforeParse(win) {
        win.document.cookie = 'session=abc; token=xyz';
        win.__customState = { ts: Date.now() };
        console.log('[hook] window frozen with custom state');
    },
});
```

**Structured logging** (config-driven):
```json
"logging": { "level": "debug", "output": "stderr" }
```

---

## 2. Capability Inventory

| Domain | Key Component / Path | Detection Point Countered |
|--------|---------------------|--------------------------|
| **Environment Fidelity** | `src/env/core.js` — 8 constructors with prototype chains | `instanceof` checks, prototype traversal |
| | `src/env/navigator.js` — 5 Chrome PDF plugins with MimeType back-references | `navigator.plugins.length`, `Plugin instanceof`, `for-of` iteration |
| | `src/env/utils.js` — `safefunction(fn, name, length)` | `Function.prototype.toString` → `[native code]` |
| **Deterministic Fingerprints** | `src/env/fingerprint/canvas.js` — mulberry32 PRNG, 300×150 valid PNG | `canvas.toDataURL()`, `getImageData()` consistency |
| | `src/env/fingerprint/webgl.js` — 5 GPU profiles, WebGL 1.0 + 2.0, 100+ stubs | `getParameter(VENDOR/RENDERER)`, `getSupportedExtensions` |
| | `src/env/fingerprint/audio.js` — AudioContext + AnalyserNode with seed-derived data | `getFloatFrequencyData`, oscillator frequency checks |
| **Lifecycle Hooks** | `src/env/browser.js` — `beforeParse(win)` callback before vm.createContext | Dynamic cookie injection, `$_ts` state seeding, event listener pre-registration |
| | `src/index.js` — `createBrowserFromConfig(config, overrides)` | Config-driven hook assembly, GPU profile auto-selection |
| **Behavior Transparency** | `src/env/document-all.js` — callable Proxy + `getAccessLog()` | Automated SDK probe detection, stack-trace capture per property access |
| | `src/index.js` — `Fingerprint.getProfiles()` GPU pool inspection | Pre-flight hardware profile validation |
| **Anti-Rehost** | `src/env/browser.js` — `process/require/global/module/Deno` → `undefined` | Node.js global detection |
| | `src/env/core.js` — `Illegal constructor` throws on all 8 constructors | Direct constructor invocation detection |
| **Determinism Control** | `src/env/fingerprint/canvas.js` — seed-derived pixel data | Regression testing (same seed → byte-identical signatures) |
| | `src/index.js` — `canvasSeed` option, `fingerprint.seed` in config | Production randomization (seed: "random" → Date.now()) |

---

## 3. Operational Strategy

### 3.1 Three-Step Workflow

```
Step 1: ANALYZE        Step 2: CONFIGURE        Step 3: MONITOR & RUN
┌──────────────┐       ┌──────────────────┐     ┌──────────────────────┐
│ capture_sdk  │       │ cp templates/    │     │ debugAll: true       │
│ trace_env    │  →    │ default-anti-    │  →  │ getAccessLog()       │
│ identify     │       │ detect.json →    │     │ beforeParse inject   │
│ detection    │       │ targets/<name>.json │  │ Phase 7 validate     │
│ points       │       │ edit 5 fields    │     │ iterate if 403       │
└──────────────┘       └──────────────────┘     └──────────────────────┘
```

**Step 1 — Analyze:** Run `capture_sdk.js` (Phase 1), then `trace_env.js` (Phase 2).
Identify which browser APIs the SDK touches. Note any unusual checks
(e.g., `document.all == undefined`, specific GPU vendor strings).

**Step 2 — Configure:** Copy the baseline template, override only what differs:
```bash
cp config/templates/default-anti-detect.json config/targets/<platform>.json
```
Edit the 5 most common fields:
- `target.name` → `"<platform-name>"`
- `target.test_url` → actual API endpoint
- `browser.user_agent` → match captured page
- `webgl.profile` → `"intel_integrated"` or `"nvidia_desktop"`
- `fingerprint.seed` → `42` for regression, `"random"` for production

**Step 3 — Monitor & Run:** Enable `document_all.monitoring_enabled: true`.
Run the signer. Inspect `win.document.all.getAccessLog()` to see exactly
what the SDK probed. If 403, check:
1. `document.all` log — was `typeof` checked? (needs C++ addon)
2. WebGL vendor — wrong GPU profile? → `setProfile()`
3. Canvas toDataURL — wrong seed? → check Phase 7 response body

### 3.2 Adaptation Strategy

When a new anti-bot detection is encountered, the priority is:
**never modify core → always add config.**

| Scenario | Action | File Changed |
|----------|--------|-------------|
| New GPU vendor detected | `Toolkit.Fingerprint.GPU_POOL.push({vendor, renderer, ...})` | `src/index.js` call, or `webgl.gpu_pool_override` in config |
| SDK checks `navigator.connection` | Add `connection` stub to `buildNavigator()` | `src/env/navigator.js` (one-time core change, config-gated) |
| SDK expects different fake response JSON | Override `network.fake_response_json` in config | `config/targets/<name>.json` only |
| SDK rotation (new JS bundle) | Update `sdk_path` in config, re-run Phase 2 | `config/targets/<name>.json` only |
| New target site | Copy template → edit fields → validate | New `config/targets/<name>.json` |

### 3.3 Phase Pipeline (Legacy Reference)

The 7-phase workflow from v1 remains the foundation. v2.1 adds
config-driven automation on top:

| # | Phase | v1 Action | v2.1 Enhancement |
|---|-------|-----------|-----------------|
| 1 | Acquire SDK | `node core/capture_sdk.js --url "..."` | Same |
| 2 | Trace env | `node core/trace_env.js bundles/s.js > bundles/fake_env.js` | Same |
| 3 | Build env | Hand-edit fake_env.js | **Skip — prototype engine handles this** |
| 4 | Intercept XHR | Manual sign.js wiring | **Config `network.fake_response_json`** |
| 5 | Init config | DevTools copy `.init({...})` verbatim | **Config `lifecycle.before_parse` injects $_ts** |
| 6 | Lock determinism | Seed Math.random/Date.now | **`fingerprint.seed` + `canvasSeed` option** |
| 7 | Validate | `curl_cffi` roundtrip | Same — **plus `document.all` monitoring if 403** |

---

## Reminders

- **Never skip Phase 2** — the Proxy tracer catches cold properties you'd miss from memory.
- `process`, `require`, `global`, `module`, `Deno` → `undefined` in sandbox.
- Fake XHR `responseText` must be valid JSON the SDK can parse.
- Call `process.exit(0)` after capturing signature — SDKs install timers.
- Param order in signed URLs matters — don't use `URLSearchParams` to clean them.
- **New config before new code** — always try a config override before touching `src/env/`.
- `document.all == undefined` cannot be faked in pure JS — enable monitoring first,
  then compile the C++ addon only if the target SDK checks this specific condition.
