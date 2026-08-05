# Loop History

## 2026-08-03 — Greenfield Flutter app "OpenCode Remote" (Android-first OpenCode server client)
- Tier: COMPLEX (budget extended 7 → 9 by user mid-run)
- Repair rounds used: 8 / 9 (FINAL_GATE reserve 2 untouched at all times)
- Task: production Flutter app — onboarding wizard (Basic Auth, /global/health, secure storage), home (projects + recent sessions), project screen (sessions CRUD/search), session chat (typed parts, tool cards, SSE streaming deltas, composer send/stop, permissions card, drawer, diff viewer with Myers line diff, files browser, terminal via shell endpoint, agent/model selectors, settings, offline UX) wired to the real OpenCode API (schemas verified against anomalyco/opencode dev SDK types.gen.ts + server source)
- Failed hypotheses: none (all 8 repairs were direct, evidence-backed)
- TARGETED: PASS after each chunk (final 267/267 tests, analyze clean)
- REVIEWER: APPROVED for Chunks A-E (0 CRITICAL; IMPORTANT findings repaired; deferred MINORs tracked)
- FINAL_GATE: PASS with caveat — analyze clean, 267/267, debug APK builds (app-debug.apk 173 MB), dart format now exits 0, hygiene scans clean (no hardcoded creds, no raw 401/SocketException UI strings, zero analytics, HTTPS-only remote default, password only in secure storage, manifests correct). Caveat: gate's `dart format` run was accidentally in-place and rewrote 60/91 files whitespace-only; no VCS to restore byte-exact state; reformatted tree fully re-verified green and adopted as baseline (format drift across repair rounds, LOW/cosmetic)
- Skipped checks: release APK (no signing), iOS/macOS builds, on-device/emulator + live-server SSE verification (deferred per user decision — unit/widget tests only)
- Pre-existing failures: none (greenfield)
- Deferred MINORs: pipeline shared-buffer multi-future flush race; orphaned part before message; unbounded SSE decoder buffer; controllers never closed; untested backoff-stability paths; drawer/new-route test coverage; composer agent/model pass-through not directly asserted; post-reconnect ProjectScreen one Retry tap; Open Tailscale snackbar-only; files path special-chars test
- Outcome: SUCCESS — Phase 1-3 complete, verified, reviewed. User action items: run opencode serve/web with OPENCODE_SERVER_USERNAME/PASSWORD + `tailscale serve --bg 4096` on the Mac; install debug APK on Android; connect via .ts.net URL; verify live flows (streaming, permissions, abort) on-device — real-server integration skipped per user choice

## 2026-08-02 — Change Luna model to deepseek-v4-flash for aicredits
- Tier: STANDARD
- Repair rounds used: 0 / 5 (final-gate reserve untouched)
- Failed hypotheses: none
- TARGETED: PASS (119 tests across 6 files, tsc clean, runtime resolution verified)
- REVIEWER: APPROVED (2 MINOR deferred: deepseek catch-all in modelFamilyOf could mis-tier a future terra-class deepseek ID; getFallbackModelForRole ignores AI_MODEL_LUNA family default)
- FINAL_GATE: PASS (npm test 218/218 + production build + node tests 4/4, tsc exit 0, eslint exit 0 with 8 pre-existing-style warnings incl. 3 unused imports in tests/observabilityInvocation.test.ts)
- Skipped checks: live aicredits API call (mocked suite; runtime/ops concern); no CI config exists in repo
- Pre-existing failures: none found
- Outcome: SUCCESS — DEFAULT_LUNA_MODEL="deepseek-v4-flash"; .env/.env.example Luna vars updated; modelFamilyOf maps "deepseek"→luna cost family (rates unchanged); docs + 2 test files updated

## 2026-08-02 — Fix trace a0e0d331: deepseek-v4-flash MALFORMED_JSON failures + gemini auth rejections
- Tier: COMPLEX
- Repair rounds used: 0 / 7 (final-gate reserve untouched)
- Failed hypotheses: none
- Root cause: (1) DEFAULT_LUNA_MODEL="deepseek-v4-flash" (changed same day WITHOUT live validation) returned unparseable output 10/10 attempts via aicredits while openai/gpt-5.6-terra succeeded; aicredits fallback rung was skipped because getFallbackModelForRole returned the same model as primary. (2) GEMINI_API_KEY present but rejected by Gemini API (classifier auth branch: PROVIDER_UNAVAILABLE + PROVIDER_REJECTED_REQUEST) — environment issue, cannot fix in code. (3) Latent bug: chat-completions SSE path never checked finish_reason, so streamed truncation was misreported as MALFORMED_JSON instead of retryable OUTPUT_TRUNCATION.
- Fix: DEFAULT_LUNA_MODEL → "openai/gpt-5.6-luna"; Luna fallback → AI_MODEL_LUNA_FALLBACK env || "openai/gpt-5.6-terra" (verified-working model, activates distinct aicredits fallback rung); SSE finish_reason max_tokens/length → ProviderAttemptError(stopReason max_tokens) before parseJsonText; .env/.env.example Luna vars aligned (AI_MODEL_* names only, no secrets); reliability test expectation updated; new SSE truncation test added; ARCHITECTURE.md updated
- TARGETED: PASS (94/94 targeted, 219/219 full vitest, tsc exit 0, scoped lint clean)
- REVIEWER: APPROVED (7 MINOR deferred: cost attribution uses requestedModel on fallback success; truncated attempts omitted from usageSink; non-stream finish_reason "length" unchecked; test hygiene on failing-path ladder rungs; .env.example gitignored so example changes untracked; unused modelRoleUsesTerraFallback export; stage-level preciseFailureCategory takes last rung)
- FINAL_GATE: PASS (npm test 219/219 + production build + node tests 4/4, tsc exit 0, eslint exit 0 with same 8 pre-existing warnings, no scope creep)
- Skipped checks: live aicredits API call (requires credentials; mocked suite); no CI config exists in repo
- Pre-existing failures: none found
- Deferred: prompt-bloat redesign (context_critical 402%/301%/287%/193% — real design flaw but NOT the failure cause; critic failed within budget; changing prompt content carries quality-regression risk)
- Outcome: SUCCESS — Luna pipeline now uses gpt-5.6-luna (proven family) with gpt-5.6-terra fallback; deepseek remains env-selectable; streamed truncation now classified correctly. User action needed: set a valid GEMINI_API_KEY (current key rejected by Gemini API).

## 2026-08-02 — Rename Luna/Terra tiers to Cheap/Premium; cheap model = deepseek-v4-flash
- Tier: COMPLEX
- Repair rounds used: 2 / 7 (final-gate reserve untouched)
- Failed hypotheses: none
- Scope: user asked to (1) set the Luna model to deepseek-v4-flash for aicredits, (2) replace Luna/Terra labels with cost-based tiers (Cheap/Premium), removing AI_MODEL_LUNA/AI_MODEL_TERRA env names.
- Implemented: DEFAULT_CHEAP_MODEL="deepseek-v4-flash", DEFAULT_PREMIUM_MODEL="openai/gpt-5.6-terra"; CHEAP_ROLES; env AI_MODEL_CHEAP/AI_MODEL_PREMIUM/AI_MODEL_CHEAP_FALLBACK/AI_MODEL_PREMIUM_FALLBACK; modelFamilyOf "cheap"|"premium" (detects luna/terra/deepseek tokens, same rates); buildPremiumEvidencePacket/PremiumEvidencePacket; repairClass CHEAP_REPAIR/PREMIUM_REPAIR; audit cheapCallCount/premiumCallCount; MAX_PREMIUM_CALLS (8→32, legacy MAX_TERRA_CALLS fallback read); debug API + UI Cheap/Premium labels with legacy luna/terra key fallbacks; .env/.env.example updated (AI_MODEL_* names only, no secrets); docs updated (zero Luna/Terra remaining); 9 test files + 2 new test files.
- Repair 1 (reviewer): workspaceService.ts old persisted rows (repairClass LUNA_REPAIR/TERRA_REPAIR) crashed hydration via strict zod parse → parsePersistedProductReality + remapLegacyRepairClass + tests; pipelineBudget premiumCallCount role-based mislabel → model-based classification via modelFamilyOf(actualModel ?? requestedModel).
- Repair 2 (reviewer): model-based counting + MAX_PREMIUM_CALLS=8 guard would trip mid-run when cheap model is broken (all cheap-role calls fall back to premium) → MAX_PREMIUM_CALLS raised to 32 (≈2× worst-case ~16 premium invocations, < MAX_PIPELINE_AI_CALLS=60, spend bounded by MAX_PIPELINE_COST=$5) + tests/pipelineBudget.test.ts (5) + docs note + .env.example knob comment fixed to 32.
- TARGETED: PASS (233/233 full vitest, tsc exit 0, scoped lint clean; repair verifications independently re-run)
- REVIEWER: APPROVED (3rd pass; 0 CRITICAL/IMPORTANT; MINOR: .env.example comment value fixed; deferred: MAX_TERRA_CALLS legacy fallback keeps tight cap if set, modelFamilyOf substring catch-all, MAX_PIPELINE_COST hard cap on fallback runs)
- FINAL_GATE: PASS (npm test 233/233 + production build + node tests 4/4, lint exit 0 same 8 pre-existing warnings, tsc exit 0, no scope creep, final luna|terra grep clean — only historical fixture strings, legacy back-compat reads, detection tokens, premium model ID)
- Skipped checks: live aicredits/Gemini calls (need credentials; mocked suite); no CI config exists in repo
- Pre-existing failures: none found
- Deferred MINORs: modelFamilyOf substring detection brittle for future model IDs; unknown-family models count cheap; MAX_PIPELINE_COST can abort full-fallback runs by design; .env.example gitignored (`.env*` rule) so example changes are local-only
- User action items: GEMINI_API_KEY is still a placeholder/rejected — set a valid key for the Gemini rung; deepseek-v4-flash is the cheap-tier default again (its MALFORMED_JSON history is mitigated by the premium fallback rung: ladder [deepseek, gpt-5.6-terra, gemini], 2 attempts per rung)
- Outcome: SUCCESS — tier labels renamed Cheap/Premium everywhere (code, env, UI, docs, tests); cheap tier = deepseek-v4-flash with premium fallback; old persisted audit rows hydrate via legacy remap; budget guard sized for fallback-heavy runs.

## 2026-08-02 — Fix trace 58b31549 total pipeline failure (deepseek-v4-flash reasoning truncation + missing schema); models unchanged
- Tier: COMPLEX
- Repair rounds used: 2 / 7 (final-gate reserve untouched)
- Failed hypotheses: none
- Root cause (from trace + live probes): (1) deepseek-v4-flash is a REASONING model (streams reasoning_content) but received the small per-role output caps (2000/3000/1500/3500) — chain-of-thought consumed the budget → finish_reason max_tokens → TRUNCATED_OUTPUT 10/10 attempts, all stages; (2) the JSON schema was never sent to OpenAI-compatible models (only the strategist embedded it) — the openai/gpt-5.6-luna fallback produced wrong-shape JSON (SCHEMA_VALIDATION_FAILED 6/6); (3) stage timeouts (90s/120s) too tight for deepseek latency; (4) extractor cap 2000 too small for 45-atom reviews; (5) gemini key invalid (environment) + fallback rung skipped in .env (AI_MODEL_CHEAP_FALLBACK=deepseek, luna line commented) → total failure at required strategist stage.
- Fixes: usesReasoningModel = usesOpenAiReasoning || /deepseek/i → reasoning models get 32K output budget (was 16K openai-only); JSON schema embedded in extractor/canonicalizer/critic/targetedRepair system prompts (strategist pattern; versions bumped *-v2.2); ROLE_MAX_OUTPUT_TOKENS.EVIDENCE_EXTRACTOR 2000→4000; stage timeouts extractor/canonicalizer/strategist 300s, critic 360s, repairs 120/240s; TOTAL_PIPELINE_TIMEOUT_MS 15→30 min; AICREDITS_TIMEOUT_MS 300→420s; all 7 stage invoke sites now pass STAGE_TIMEOUT_MS (strategist/critic/repair previously omitted).
- Live verification (user-requested): 3 live full-pipeline runs via scripts-live harness (isolated from default suite; real aicredits calls, deepseek only). Final run: PIPELINE SUCCESS — all 6 invocations succeeded attempt 1 (extractor 97.5s, canonicalizer 2×120s, strategist 175s, critic 81s, repair 6.7s), parseStatus ok everywhere, 0 truncations, report produced (verdict + 3 recs/3 exp/3 scripts), 8.0 min runtime. One flake run: canonicalizer batch hit 240s under latency variance (budget since raised to 300s).
- TARGETED: PASS (235/235 vitest, tsc exit 0, lint baseline; final state independently verified)
- REVIEWER: APPROVED (0 CRITICAL/IMPORTANT blocking; IMPORTANT pre-existing: MAX_PIPELINE_COST never enforces — providerCost always null; MINOR deferred: repair fit-check uses SIMPLE_REPAIR for STRATEGIC_REPAIR, gemini rung lacks timeout, role-budget bypass inflates cost estimate)
- FINAL_GATE: PASS (npm test 235/235 + production build + node tests 4/4, lint exit 0 same 8 pre-existing warnings, tsc exit 0, offline confirmed — no live calls during gate, scope clean)
- Skipped checks: none material (live suite not re-run in gate — paid calls; live evidence from subtask run)
- Pre-existing failures: none found
- Deferred MINORs: /deepseek/i over-matches future non-reasoning deepseek variants (low risk: max_tokens is a cap not a spend target); MAX_PIPELINE_COST is observability-only (cost bounded by call counts); repair fit-check timeout mismatch; gemini rung no timeout (masked by invalid key); test-deepseek.mjs + raw-md-loader.mjs harness housekeeping
- User action items: (1) GEMINI_API_KEY in .env is invalid ("API key not valid") — every gemini fallback fails instantly; fix or remove the rung. (2) .env has AI_MODEL_CHEAP_FALLBACK=deepseek-v4-flash active and the openai/gpt-5.6-luna fallback line COMMENTED OUT — pipeline is effectively single-model/single-provider; uncomment the luna/terra fallback to restore a real fallback rung. (3) Security: reviewer retracted tester's claim — .env is NOT committed (gitignored, never in history); no key in repo.
- Outcome: SUCCESS — deepseek-v4-flash now works end-to-end with the system (reasoning budget + schema visibility + timeouts); pipeline completes with all stages semantic_review; residual risk is aicredits latency variance (canonicalizer 300s budget) and the dead gemini/fallback config (user action).

## 2026-08-02 — Fix trace 94d6a4d2: canonicalizer TIMEOUT from prompt bloat + env/billing issues
- Tier: COMPLEX
- Repair rounds used: 1 / 7 (final-gate reserve untouched)
- Failed hypotheses: none
- Root cause: with a SUCCESSFUL extraction (45 atoms / 58 observations), the canonicalizer embedded the FULL extraction review + FULL source-coverage audit in EVERY batch's system prompt (78.5K chars, ~19.7K tokens) → deepseek-v4-flash took >360s → all 3 batches aborted at the 300s stage timeout → deterministic_only degradation. Plus environment: aicredits credit balance exhausted mid-run (HTTP 402 at strategist) and gemini key invalid.
- Fixes: canonicalizer prompt per-batch filtering (EXTRACTION REVIEW observations by batchEvidenceIds; omissions + SOURCE COVERAGE AUDIT entries by batchSourceIds) + hard 12K per-section caps via boundedJson (front-keep/tail-drop + {"__truncated":true} marker); CANONICALIZATION_PROMPT_VERSION → v2.3; STAGE_TIMEOUT_MS.CANONICALIZER 300→480s and AICREDITS_TIMEOUT_MS 420→480s (deadline math 300+480+300+360 = 1440s = 24 min < 30 min); boundedJson comment fix. validatedReview still validates gaps against the FULL audit (filtered/truncated prompt sections only limit proposals, never validate).
- Live verification: 48-atom harness run — canonicalizer 3/3 batches SUCCEEDED (52-97s with fresh credits; 382-396s in a credit-exhaustion-window run — headroom now 480s). Final run with topped-up credits: PIPELINE SUCCESS — 7/7 invocations (extractor 135s, canonicalizer 3×52-97s, strategist 124s, critic 71s, repair 9s), 0 failures/retries/HTTP errors, report produced (3 recs/3 exp/3 scripts, PREMIUM_REPAIR used), 7.25 min runtime.
- TARGETED: PASS (237/237 vitest, tsc exit 0, lint baseline; final state independently verified)
- REVIEWER: APPROVED (re-pass; 0 CRITICAL/IMPORTANT; deferred: deadline comment overstates "after each stage" enforcement, MAX_STAGE_INPUT_CHARS guard checks input only, boundedJson comment phrasing)
- FINAL_GATE: PASS (npm test 237/237 + production build + node tests 4/4, lint exit 0 same 8 pre-existing warnings, tsc exit 0, offline confirmed, scope clean)
- Skipped checks: live suite not run in gate (paid calls; live evidence from the 7/7 run)
- Pre-existing failures: none found
- Deferred MINORs: ATOMIC EVIDENCE section uncapped in pathological single-observation cases; possible duplicate coverage gaps across batches; MAX_STAGE_INPUT_CHARS guard doc mismatch (input-only); strategist MALFORMED_JSON flake observed once (retryable, model-output variance — retry handles it)
- User action items: (1) aicredits credits were exhausted mid-run (HTTP 402) — user topped up, confirmed resolved. (2) GEMINI_API_KEY still invalid — gemini rung is dead weight until fixed. (3) AI_MODEL_CHEAP_FALLBACK=deepseek-v4-flash == primary → fallback rung skipped; uncomment the openai/gpt-5.6-luna line for a real fallback.
- Outcome: SUCCESS — canonicalizer prompt bloat eliminated (per-batch filtering + 12K caps), 480s latency headroom, pipeline verified end-to-end 7/7 with deepseek-v4-flash on all stages; residual risks: aicredits latency variance (480s ceiling), dead gemini/fallback config (user action).

## 2026-08-02 — Fix trace 1cfabb22: strategist TIMEOUT at 300s stage cap
- Tier: COMPLEX
- Repair rounds used: 0 / 7 (final-gate reserve untouched)
- Root cause: latest app run — extractor + canonicalizer (3/3, prompt-bloat fix live: prompts 66K/64K/42K) succeeded; STRATEGIST aborted at exactly 300s (its stage timeout, never raised in the earlier rounds) under aicredits latency variance (healthy run: 124s; this run: >300s; all stages ~40-60% slower that run).
- Fix: STAGE_TIMEOUT_MS.STRATEGIST 300_000 → 480_000 (matches canonicalizer + transport ceiling AICREDITS_TIMEOUT_MS); comments updated (worst-case one-attempt success path 300+480+480+360 = 1620s = 27 min < 30-min deadline; strategist runs unconditionally without a deadline fit-check — pre-existing, documented). TIMEOUT is not retryable, so a timing-out strategist moves straight to the gemini rung (max +180s added to failure latency).
- TARGETED: PASS (237/237 vitest, tsc exit 0, lint clean)
- REVIEWER: APPROVED (0 CRITICAL/IMPORTANT; MINOR deferred: transport ceiling comment mentions only CANONICALIZER as largest; docs/AI_PIPELINE.md "300-second request window" stale; STRATEGIST output cap experiment deferred)
- FINAL_GATE: PASS (npm test 237/237 + production build + node tests 4/4, lint exit 0 same 8 pre-existing warnings, tsc exit 0, offline confirmed, no scope creep)
- Skipped checks: live suite not run in gate (paid calls)
- Pre-existing failures: none found
- User action items (unchanged): GEMINI_API_KEY invalid — gemini rung dead weight; AI_MODEL_CHEAP_FALLBACK=deepseek-v4-flash skips the fallback rung — uncomment the openai/gpt-5.6-luna line for a real fallback.
- Outcome: SUCCESS — all stages now have latency headroom up to the 480s transport ceiling; the session's four failing traces (a0e0d331, 58b31549, 94d6a4d2, 1cfabb22) all addressed: reasoning budget 32K, schema embedding, canonicalizer prompt caps, stage timeouts 300/480/480/360/120/240. User should re-run the app evaluation for a clean trace.
