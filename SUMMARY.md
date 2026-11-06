# dsh-research-report — Development Delivery Summary

## Implementation Checklist

**Capability Seams (Single package, three roles)**
- Service Definition: `src/service.ts` — `ResearchReportService extends Service` (`ctx.researchReport`), containing the byte-frozen `assemble(request: AssembleReportRequest): Promise<AssembleReportResult>` contract block (`ReportSectionInput`/`EvidenceInput`/`AssembleReportRequest`/`AssembleReportResult`), as well as internal methods `addEvidence`/`verifyClaim`/`getEvidence`/`readEvidenceContent`/`getClaim`/`listEvidence`/`listClaims`/`summarize`. `scripts/verify-frozen-contract.mjs` enforces byte-level gates on two frozen text blocks (the assemble contract + `CitationCheckRequest`/`CitationCheckResult` bridge).
- Provider: `src/provider-local.ts` — `LocalResearchReportService`: policy limits (`maxEvidenceBytes`/`maxEvidencePerReport`), ledger orchestration, verification verdict writeback, seal directory allocation (`<reportRoot>/<slug(topic)>/<YYYYMMDD-HHmmss>/`, appending numeric index on same-second collisions).
- Consumer: `src/tools/` — three `defineTool` definitions: `evidence_add`, `research_report`, and `ledger_query`.

**Evidence Ledger (`src/ledger.ts`, pure Node with zero DSH dependencies)**
- Layout: `<ledgerRoot>/objects/<sha256>` (immutable snapshots committed via atomic tmp+rename) + `index.jsonl` (id→hash/origin/title/capturedAt/bytes) + `claims.jsonl` (claim→evidenceIds binding, immutable registration) + `verdicts.jsonl` (verdict writeback, latest takes precedence).
- Content deduplication (default id is `ev-<hash12>`); loud rejection on identical id with differing content (`ID_CONFLICT`); recomputes hash on read → `ok`/`tampered`/`missing`; loud failure on corrupted JSONL; writes serialized through internal queue.

**Verification (`src/verify.ts`)**
- Level 1 (Built-in byte-level): Numbers and citation substrings (`"…"`, `“…”`, `「…」`) in claim text must be literally locatable in bound snapshots. Numbers adopt presence-first: locatable confirms support; if claimed value is **missing** and its left context label corresponds to a **different** numeric value in snapshot → `contradicted`; if neither locatable nor contradicting information found → `unverified`; if no verifiable literals exist → `unverified`.
- Level 2 (Optional numeric verification bridge): When claim includes `dataset` + `citations` and `ctx.get('dataQuality')` is present, tolerance checking occurs via frozen contract `verifyCitations` (mismatch → contradicted, not-found/unverifiable → unverified); when absent, Level 1 only with explicit note. Never imports foreign package, never writes to inject.

**Assembly and Sealing (`src/assemble.ts`, pure function)**
- Loud validation failure (`RequestValidationError`): incomplete claim citations, unknown evidence id, duplicate ids, exceeding limits, empty title/topic, unparseable `capturedAt`.
- `report.md`: Title/topic/generation timestamp/conclusion statistics + section body text (unverified/contradicted claims retain prominent inline markers `[UNVERIFIED]`/`[CONTRADICTED]` in cited paragraphs) + Appendix A verification table (claim|verdict|evidence|note) + Appendix B evidence catalog (with origin and sha256) + Appendix C sealing info (with configuration fingerprint) + Appendix D disconfirmation log.
- `manifest.json`: Fixed key-order JSON (schema `dsh-research-report/v1`, report hash, evidence hash list, claims, verdicts, config fingerprint, generator version); `sealHash` = SHA-256 of manifest bytes, recomputable and deterministic.

**Capture (`src/gather.ts`)**
- URL sources fetched via `ctx.web.fetch` (timeouts linked between `fetchTimeoutMs` and `exec.signal` via `AbortSignal.timeout/any`); local sources via `node:fs`, with dual `path.resolve` workspace escape checks.
- `gather` candidate round: `ctx.web.search` → fetch and register source by source; uncaptured sources listed in gaps catalog with failure reasons; never auto-assembles draft.

**Tools**
- `evidence_add({origin, content?, title?})` → `{ok:true, evidenceId, hash, bytes, title, origin, capturedAt, deduplicated}` or `{ok:false, error:{code,message}}`; throws when URL source used without `ctx.web` (`isError` loud failure).
- `research_report({topic, title?, sections, claims, evidenceRefs, gather?, depth?, background?})` → three specification returns: `sealed` (`reportDir`/`reportFile`/`manifestFile`/`sealHash`/`verdicts`/`counts`/`evidenceCount`) | `background` (`{kind:'background', jobId}`, `research-report` job type merged via `JobKindMap`, real `ctx.jobs`) | `gathered` (candidates + gaps). `presentResult` declares `report.md` / `manifest.json` as Web UI deliverables via `kind:'edit'` + `locations`.
- `ledger_query({claimId?|evidenceId?})` → `evidence` (with real-time integrity) | `claim` (with latest verdict) | `summary` | `not-found`; read-only.

**Session Events (`SessionEventMap` declaration merging, typed + JSDoc @mode/@param)**
- `research-report/evidence`, `research-report/verify`, `research-report/seal`. In rc.6, `Session.append` has no `ignorable` flag and no plugin event registration surface; appending unknown types causes the persistence layer to reject the log during restoration. Thus an adaptive pattern (proven in dsh-memento) is adopted: only append when `KNOWN_SESSION_EVENT_TYPES` recognizes the type. The ledger JSONL is always the authoritative persistence layer; events will auto-activate once the host records the vocabulary in the future.

**Configuration (Schemastery, `src/config.ts` + `cordis.patch.yml` annotations + five-language README table)**
- `enabled` (true), `ledgerRoot` (.research-ledger), `reportRoot` (research-reports), `maxEvidenceBytes` (2097152), `maxEvidencePerReport` (200), `fetchTimeoutMs` (20000); `resolveConfig` explicitly validates boundaries, failing loudly during load time.

**Service Inject**: `['tools', 'systemPrompt']`; `web`/`jobs`/`dataQuality` are all checked for nullness at invocation time via `ctx.get` (can mount when absent; affected paths fail loudly). Injected prompt begins with a one-sentence role statement, two sentences total.

## Acceptance Items

1. **Check Chain All Green** (Windows 11 + Node v22.22.3 + pnpm 11.7.0):
   - `pnpm install` ✅ (standalone pnpm-lock.yaml)
   - `pnpm run typecheck` ✅ exit 0 (src + test dual tsconfig)
   - `pnpm run typecheck:ci` ✅ exit 0 (skipLibCheck=false + verbatimModuleSyntax, checked against published 0.1.0-rc.6 types)
   - `pnpm test` ✅ **66/66** (8 spec files)
   - `pnpm run test:coverage` ✅ aggregate statements 92.52% / branches 81.71% / functions 95.87% / lines 92.52% (thresholds 90/80/90/90)
   - `pnpm run build` ✅ (tsdown + tsc declarations + fix-dts, zero `.ts` residual imports)
   - `pnpm run verify:self-contained` ✅ (no foreign workspace package specifiers)
   - `pnpm run verify:artifacts` ✅ (syntax check + pure Node ESM import + plugin surface + frozen service export)
   - `pnpm run verify:frozen-contract` ✅ (both frozen text blocks byte-verbatim in src/)
   - `node scripts/check-readme-sync.mjs` ✅ (five-language heading structure + config table keys match)
   - `pnpm run pack:check` ✅ (tarball includes lib/src/cordis.patch.yml/CHANGELOG/five-language READMEs/LICENSE/THIRD_PARTY_NOTICES)
   - `pnpm run lint` ✅ oxlint 0 warnings/0 errors (32 files)
2. **Test Coverage** (Real Context/Session/ToolRuntime/LocalJobRegistry/WebRuntime, only network backend mocked via scripted provider registered in real `ctx.web` registry): ledger content addressing and deduplication (`test/ledger.spec.ts`), tamper detection (modifying objects file → integrity tampered → contradicted), byte-level tri-state verification (`test/verify.spec.ts`), incomplete claim citations/unknown evidence loud rejection (`test/assemble.spec.ts` + `test/index.spec.ts`), manifest/sealHash recomputable and deterministic, loud failure when `ctx.web` is absent for URL sources (isError + WEB_UNAVAILABLE), background jobs routed via real ctx.jobs (`research-report-N`).
3. **End-to-End (Keyless)**: `test/e2e.spec.ts` — 3 local markdown documents from fixtures (workspace-relative paths) → `evidence_add` ×3 → `research_report` (topic "sample-industry-overview") → versioned seal directory (`reports/sample-industry-overview/<YYYYMMDD-HHmmss>/`) → Appendix A three rows ✅ verified → manifest hash recomputation == sealHash → tamper with one evidence object → `ledger_query` reports `integrity: tampered` → re-run assemble → `c-growth` judged `contradicted` and body text contains `[CONTRADICTED]`.
4. **Trial Installation**: Manual temporary profile (`DSH_HOME` inside this plugin's `.tmp/`, cleaned up) — `dsh plugin --profile smoke add dsh-base + dsh-headless + this package tgz` ✅ → `--dump-config` contains `dsh-research-report` line ✅ → keyless headless smoke test yields `MISSING_CREDENTIAL` (proves full plugin tree mounted) ✅ → `remove` cleanly uninstalls ✅. (dsh-test-drive itself requires installing into a running host profile to drive the target, so an equivalent manual step matrix was used.)
5. **Frozen Contract Self-Check**: `scripts/verify-frozen-contract.mjs` embeds §6.2 and §7 texts and asserts byte-verbatim existence in `src/service.ts`/`src/verify.ts` ✅; `ctx.researchReport.assemble(request)` matches frozen signature (provider implementation only adds one optional `context` param, assignment-compatible with frozen surface).

## Deviations & Notes (Prompt vs Official Seams/Templates)

- **`ctx.attachment` does not exist in required form in rc.6**: The official attachment seam is `ctx.attachments` and **only supports images** (PNG/JPEG/WebP/GIF); cannot output markdown/manifest. In rc.6, the real mechanism for "deliverables appearing in Web UI deliverables" is tool cards with `locations` (diff card or `kind:'edit'` generic card). Implemented via real mechanism: `research_report`'s `presentResult` uses `presentationMeta` (persisted in `tool/result.meta`, replay-safe) to declare `report.md`/`manifest.json` as deliverable paths. Report files are written directly into workspace (`<reportRoot>/...`).
- **Adaptive Session Events**: Task required "recording session events", but in rc.6 appending unrecorded event types causes session log restoration to fail (both dsh-fast and dsh-memento verified this issue). Declared fully typed events with adaptive appending; ledger JSONL remains the authoritative persistence layer.
- **Default profile lacks fetch provider**: `dsh-base` only mounts search (fetch disabled by default for SSRF concerns). URL fetching therefore fails loudly in default deployment (`WEB_UNAVAILABLE`/`WEB_PROVIDER_UNAVAILABLE`, documented in README), and `gather` places uncaptured sources into the gaps list instead of fabricating evidence.
- **Directory Naming**: Uses family-standard `test/` (task prompt wrote `tests/`) and five-language READMEs (English is source of truth).

## Known Limitations

- Byte-level verification is **non-semantic**: paraphrased claims without literal numbers/citations → `unverified`; label conflict detection is deterministic heuristic (presence-first, only judged contradicted when claimed value is missing), without full-text semantic embedding alignment (explicitly out of scope for v1).
- Session events are declarative in rc.6 (persisted once host incorporates vocabulary).
- Single-workspace scope: ledger/report roots resolve relative to harness cwd at mount time; multi-workspace deployments need absolute paths per profile.
- Snapshots stored as UTF-8 text; binary sources (PDF, etc.) not included in v1.
- Background job tests run in unowned mode (minimal harness agent is not registered in real agents registry; production profiles obtain agent via agent loop, which is standard path).

## Future Recommendations

1. Remove `KNOWN_SESSION_EVENT_TYPES` adaptive gate once host supports `ignorable` or plugin event registration surface (event definitions are already complete).
2. Once fetch provider is included in default deployment, `gather` candidate capture rate will naturally increase; add a deployment configuration example to README.
3. For v2: evaluate citation location enhancements (more quote styles / normalized cross-paragraph numbers), heuristic CSV column auto-mapping when `ctx.dataQuality` hits, and report directory attachment integration (when official attachment seam supports generic files).
4. Release execution followed PHASE2-GROUP-PROMPTS.md §0.3 handoff chain.

## Release Handoff (§0.3 Chain Execution Status)

**Credentials Boundary (Truthful Record)**: Windows Credentials Manager held active GitHub credentials; git push and gh CLI executed transparently under logged-in account PerryLink (token masked throughout); npm publish executed via release workflow (NPM_TOKEN secret).

### Step 0 · Community Feedback Check
- Repository had 0 issues / 0 PRs → no unreplied community comments.
- Discussions enabled, welcome thread #1 posted.
- Outgoing PR status: awesome-dsh-plugin [PR #1990](https://github.com/awesome-dsh-plugin/awesome-dsh-plugin/pull/1990) (open); AdamPlatin123 [PR #260](https://github.com/AdamPlatin123/awesome-dsh-plugins/pull/260) (merged).
- No bug reports requiring fix.

### Step 1 · Standard Component B Status
- Topics (9 including `dsh-plugin`) ✅; About description ✅; homepage→npm ✅.
- Badges (License/DSH/Node/CI/Version/npm version/npm downloads) ✅; Contributors section ✅.
- P0: `.github/ISSUE_TEMPLATE/bug_report.yml`, `feature_request.yml`, `PULL_REQUEST_TEMPLATE.md`, `SECURITY.md` present ✅.
- Discussions enabled + Welcome thread ✅.
- Main branch protection verified: `required_status_checks: {strict:false, contexts:["gates"]}`.

### Step 2 · Standard Component A: Ecosystem Submission Status

Stage 0 compliance self-check: public ✅, `dsh-plugin` topic ✅, `dsh.bundle.patch`→cordis.patch.yml ✅, real code + LICENSE + README ✅, npm published ✅.

| Target | Status | Deliverable |
|---|---|---|
| I-1 awesome-dsh-plugin | PR #1990 open pending merge | Track merge |
| I-2 AdamPlatin123 | ✅ Included (PR #260 merged) | None |
| I-3 0xsline/awesome-deepseek-harness | ✅ PR #422 created (Output & Deliverables) | Patch archived in `release-handoff/0xsline-dsh-research-report.patch` |
| I-4 bruc3van self-nomination | Blocked by rule: requires `stargazers_count > 10` | Resubmit when stars > 10 |
| II (OMDSH hub) | Issue #78 created; bot precheck passed Harness plan | Manifest + diagnosis comment in Issue |
| III (Official Discussions) | ✅ Thread #3447 published (Show Your Plugins!) | None |
| IV (Aggregator repos) | Synced via daily snapshots | Follow up commands |

### Step 3 · Standard Component C: Release Status

- `main` branch pushed ✅.
- tags `v0.1.0` ✅ and `v0.1.1` ✅.
- npm: `0.1.0` ✅ and `0.1.1` ✅ with provenance.
- GitHub Releases `v0.1.0` ✅ and `v0.1.1` ✅.
