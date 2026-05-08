# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **browser environment patching toolkit** for crawler reverse engineering. The core technique: build a fake browser environment (`window`, `document`, `navigator`, etc.) inside Node's `vm` sandbox, load the target's obfuscated JS signing bundle as-is, and capture its output — producing gating signatures offline without a headless browser.

Full documentation and the 7-phase workflow live in [README.md](README.md). This file covers architecture, commands, and constraints that aren't obvious from reading individual files.

## Commands

```bash
# Phase 1: Auto-capture JS bundles from a live page
npx playwright install chromium                                     # one-time setup
node core/capture_sdk.js --url "https://www.<target>.com/"          # basic capture
node core/capture_sdk.js --url "..." --timeout 15000 --min-size 30000 --screenshot

# Phase 2: Self-healing trace (auto-generates env stub on stdout)
node core/trace_env.js bundles/signer.js > bundles/fake_env.js
node core/trace_env.js --max-rounds 12 --init bundles/signer.js     # more rounds + XHR probe

# Phase 7: Validate — must use curl_cffi, not requests/urllib
pip install curl_cffi
python -c "import curl_cffi.requests as cr; r = cr.get(url, impersonate='chrome120'); print(r.json())"

# Persistent signer from Python
SIGN_JS=./core/sign.js TARGET_URL=https://... python core/persistent_signer.py
```

There is no `package.json`, build step, or test suite — this is a template/reference kit, not an installable package.

## Architecture: data flow

The 4 core files form a pipeline aligned with the 7 phases:

```
capture_sdk.js               trace_env.js                 fake_env.js
(Phase 1)                    (Phase 2+3)                  (Phase 3 manual alt)
     │                             │                           │
     │ captured JS bundles         │ auto-healing Proxy        │ ~400-line typed stub
     │ saved to bundles/           │ traces all property       │ for vm.createContext
     │ + manifest.json             │ accesses, then            │
     │                             │ generates env stub        │
     │                             │ on stdout                 │
     ▼                             ▼                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      core/sign.js  (customize this template)      │
│  require()s the env stub, loads SDK in vm.runInContext(),       │
│  calls sign(), outputs JSONL to stdout                          │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    │ JSONL stdin/stdout
                                    ▼
                          persistent_signer.py
                          (Phase 7 bridge)
```

**Key architectural decisions:**
- `trace_env.js` auto-generates stubs via heuristics in `guessDefault()` (~80 property name patterns). It wraps every object in recursive Proxy handlers (`makeHealer`, depth=2), auto-creates missing properties on access, and re-runs the SDK up to `--max-rounds` (default 8) when it crashes. This replaces the traditional 3-5 round manual iteration.
- `fake_env.js` is the manual alternative — a static ~400-line typed stub you edit by hand. Use it when you want full control or when the auto-healer generates wrong-typed stubs that cause silent failures.
- `persistent_signer.py` spawns `node core/sign.js --server` once and reuses via JSONL protocol. First sign ~1.5s (SDK load), subsequent signs ~10-50ms. Thread-safe (`threading.Lock`). The Node side expects: startup → write `{"ready":true}` to stdout → loop reading stdin lines, replying with `{"id":N,"ok":true,"signed_url":"..."}`.
- `capture_sdk.js` filters JS responses by size (`--min-size`, default 20KB) and URL regex (`--pattern`). It writes `bundles/manifest.json` with URL/size/sha256/timestamp per captured file. Cookies passed via `--cookie` are split on `;` and mapped to the target domain.

## Phase tracking

When helping a user reverse-engineer a JS-gated endpoint, track progress through phases 1-7 (see README for full descriptions). The most load-bearing and error-prone points:

1. **Phase 2 is mandatory** — never skip tracing and write stubs from memory.
2. **Phase 5 (init config) is the #1 silent failure** — signature looks valid (right length, right alphabet) but server rejects. Copy `.init({...})` verbatim from DevTools Sources; never invent config values. Paths are typically regex prefixes (`'^/api/v1/'`), not literal strings. Always set `debug: false` explicitly.
3. **Phase 7 must use `curl_cffi`** — default `requests`/`https` fail at TLS fingerprint layer. Only a real HTTP roundtrip returning real data proves correctness.

## Key technical constraints

These are non-obvious and will waste hours if missed:

- **TLS impersonation is mandatory**: Use `curl_cffi` (Python) `impersonate="chrome120"`, `cycletls` (Node), or `utls` (Go).
- **Param order matters**: Signers hash the raw query string. Never reorder params with `URLSearchParams`.
- **Node globals must be cleared**: `process`, `require`, `global`, `module`, `Deno` → `undefined` in the sandbox.
- **Fake response JSON matters**: If the signer POSTs during init to fetch a runtime token, your fake XHR's `responseText` must contain valid JSON the SDK can parse. Generic shapes: `{"data":{"d":""},"message":"success"}`, `{"code":0,"data":null,"msg":"ok"}`.
- **Signature length varies by input URL**: Don't filter shorter/longer outputs as "wrong branch" without proof.
- **Node process won't exit**: SDKs install internal timers. Always call `process.exit(0)` explicitly after capturing the signature.
- **`Promise` identity**: If the SDK checks `fetch(...).constructor.name === 'Promise'`, your stub must return a real Promise, not a thenable.
- **Determinism locking must happen before SDK load**: SDKs cache references to `Date.now`, `Math.random`, etc. at load time.
- **Lock determinism only for regression testing** — production signers need real timestamps (servers check freshness).

## Dependencies

- **Node.js >= 18** — `vm.createContext`, `crypto.randomFillSync`
- **Playwright** — `capture_sdk.js` (install chromium separately)
- **Python >= 3.10** — `persistent_signer.py` (`str | Path` syntax)
- **curl_cffi** — Phase 7 TLS-impersonated HTTP validation

No package.json or setup.py — copy the core files you need into your project.
