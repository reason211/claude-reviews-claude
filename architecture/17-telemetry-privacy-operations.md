# 17 — Telemetry, Privacy & Operational Control: The Dark Side of Production

> 🌐 **Language**: English | [中文版 →](zh-CN/17-telemetry-privacy-operations.md)

> **Scope**: `services/analytics/` (9 modules, ~148KB), `utils/undercover.ts`, `utils/attribution.ts`, `utils/commitAttribution.ts`, `utils/fastMode.ts`, `services/remoteManagedSettings/`, `constants/prompts.ts`, `buddy/`, `voice/`, `tasks/DreamTask/`
>
> **One-liner**: The production infrastructure you don't see — dual analytics pipelines, model codename concealment, remote killswitches, and a glimpse of features waiting behind compile-time gates.

---

## Table of Contents

1. [Dual-Channel Telemetry Pipeline](#1-dual-channel-telemetry-pipeline)
2. [The Data Harvest: What Gets Collected](#2-the-data-harvest-what-gets-collected)
3. [The Opt-Out Dilemma](#3-the-opt-out-dilemma)
4. [Model Codename System](#4-model-codename-system)
5. [Feature Flag Obfuscation](#5-feature-flag-obfuscation)
6. [Undercover Mode: Concealing AI Authorship](#6-undercover-mode-concealing-ai-authorship)
7. [Remote Control & Emergency Switches](#7-remote-control--emergency-switches)
8. [The Two-Tier User Experience](#8-the-two-tier-user-experience)
9. [Future Roadmap: Evidence from Source](#9-future-roadmap-evidence-from-source)

---

## 1. Dual-Channel Telemetry Pipeline

**Source coordinates**: `src/services/analytics/` (9 files, ~148KB total)

Every tool call, every API request, every session start — each generates telemetry events that flow through a **dual-channel pipeline**. One channel stays in-house; the other reaches a third-party observability platform. Together, they form one of the most comprehensive analytics systems in any CLI tool.

### 1.1 Channel A: First-Party (Anthropic Direct)

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:300-302
const DEFAULT_LOGS_EXPORT_INTERVAL_MS = 10000    // 10-second batch flush
const DEFAULT_MAX_EXPORT_BATCH_SIZE = 200         // Up to 200 events per batch
const DEFAULT_MAX_QUEUE_SIZE = 8192               // 8K events in-memory queue
```

The first-party pipeline uses **OpenTelemetry's `LoggerProvider`** — not the global one (which serves customer OTLP telemetry), but a dedicated internal provider. Events are serialized as Protocol Buffers and shipped to:

```
POST https://api.anthropic.com/api/event_logging/batch
```

**Resilience is aggressive.** The exporter (`FirstPartyEventLoggingExporter`, 27KB) implements:
- Quadratic backoff retries with configurable max attempts
- **Disk persistence** for failed batches — events survive process crashes and are retried on the next session startup from `~/.claude/telemetry/`
- Batch configuration is remotely adjustable via GrowthBook (`tengu_1p_event_batch_config`), meaning Anthropic can change flush intervals, batch sizes, and even the target endpoint without shipping a new version

**Hot-swap safety** is a subtle engineering detail worth noting:

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:396-449
// When GrowthBook updates batch config mid-session:
// 1. Null the logger first — concurrent calls bail at the guard
// 2. forceFlush() drains the old processor's buffer
// 3. Swap to new provider; old one shuts down in background
// 4. Disk-persisted retry files use stable keys (BATCH_UUID + sessionId)
//    so the new exporter picks up any failures from the old one
```

This means the analytics pipeline can reconfigure itself on the fly — batch size, flush frequency, even the backend URL — without losing events or requiring a restart.

### 1.2 Channel B: Datadog (Third-Party Observability)

```typescript
// Source: src/services/analytics/datadog.ts:12-17
const DATADOG_LOGS_ENDPOINT = 'https://http-intake.logs.us5.datadoghq.com/api/v2/logs'
const DATADOG_CLIENT_TOKEN = 'pubbbf48e6d78dae54bceaa4acf463299bf'
const DEFAULT_FLUSH_INTERVAL_MS = 15000   // 15-second flush
const MAX_BATCH_SIZE = 100
const NETWORK_TIMEOUT_MS = 5000
```

The Datadog channel is more restrictive. Only **64 pre-approved event types** pass the whitelist filter — from `tengu_api_error` and `tengu_tool_use_success` to voice events and team memory sync signals. Every event name carries the `tengu_` prefix (more on this in §5).

**Three cardinality-reduction techniques** prevent Datadog cost explosion:

1. **MCP tool name normalization**: Any tool starting with `mcp__` gets collapsed to just `"mcp"` — preventing each unique MCP server tool from creating a new facet
2. **Model name normalization**: External users' model names are mapped to canonical names, with unrecognized models collapsed to `"other"`
3. **User bucketing**: Instead of tracking individual user IDs (which would create millions of unique facets), users are hashed into 30 buckets via `SHA256(userId) % 30`, enabling approximate unique-user alerting without cardinality explosion

```typescript
// Source: src/services/analytics/datadog.ts:281-299
const NUM_USER_BUCKETS = 30
const getUserBucket = memoize((): number => {
  const userId = getOrCreateUserID()
  const hash = createHash('sha256').update(userId).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

### 1.3 Event Sampling: GrowthBook-Controlled Volume Dial

Not every event type needs 100% capture. A **per-event sampling configuration** (`tengu_event_sampling_config`) allows Anthropic to dynamically throttle high-volume event types:

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:38-85
export function shouldSampleEvent(eventName: string): number | null {
  const config = getEventSamplingConfig()           // From GrowthBook
  const eventConfig = config[eventName]
  if (!eventConfig) return null                     // No config → 100% capture
  const sampleRate = eventConfig.sample_rate
  if (sampleRate >= 1) return null                  // Rate 1.0 → capture all
  if (sampleRate <= 0) return 0                     // Rate 0.0 → drop all
  return Math.random() < sampleRate ? sampleRate : 0  // Probabilistic sampling
}
```

This means any event type can be remotely dialed from 0% to 100% capture rate without code changes — a production-grade volume management mechanism.

> → Cross-reference: [Episode 16: Infrastructure](16-infrastructure-config.md) for GrowthBook integration details

---

## 2. The Data Harvest: What Gets Collected

**Source coordinates**: `src/services/analytics/metadata.ts` (33KB — the single largest analytics file)

Every telemetry event carries three metadata layers, assembled by `getEventMetadata()`:

### 2.1 Layer 1: Environment Fingerprint

```
// Source: metadata.ts (conceptual summary of fields)
┌─────────────────────────────────────────────────────────────────┐
│  Environment Fingerprint (14+ fields)                           │
├──────────────────┬──────────────────────────────────────────────┤
│ Runtime          │ platform, platformRaw, arch, nodeVersion     │
│ Terminal         │ terminal type (iTerm2 / Terminal.app / ...)  │
│ Dev Environment  │ installed package managers and runtimes      │
│ CI/CD            │ CI detection, GitHub Actions metadata        │
│ OS Details       │ WSL version, Linux distro, kernel version    │
│ VCS              │ version control system type                  │
│ Claude Code      │ version, build timestamp                     │
│ Deployment       │ environment identifier                       │
└──────────────────┴──────────────────────────────────────────────┘
```

This fingerprint is rich enough to identify your specific machine configuration — what OS, what terminal, what development tools, whether you're running in CI, and what VCS you use.

### 2.2 Layer 2: Process Health Metrics

```
// Source: metadata.ts (process metrics section)
┌─────────────────────────────────────────────────────────────────┐
│  Process Metrics (8+ indicators)                                │
├──────────────────┬──────────────────────────────────────────────┤
│ Timing           │ process uptime                               │
│ Memory           │ rss, heapTotal, heapUsed, external, arrays  │
│ CPU              │ usage time, percentage                       │
└──────────────────┴──────────────────────────────────────────────┘
```

### 2.3 Layer 3: User & Session Identity

```
// Source: metadata.ts (user tracking section)
┌─────────────────────────────────────────────────────────────────┐
│  User & Session Tracking                                        │
├──────────────────┬──────────────────────────────────────────────┤
│ Model            │ active model name                            │
│ Session          │ sessionId, parentSessionId                   │
│ Device           │ deviceId (persistent across sessions)        │
│ Account          │ accountUUID, organizationUUID                │
│ Subscription     │ tier (max, pro, enterprise, team)            │
│ Repository       │ remote URL hash (SHA256, first 16 chars)     │
│ Agent            │ agent type, team name                        │
└──────────────────┴──────────────────────────────────────────────┘
```

The **repository fingerprint** deserves special attention. Rather than sending the raw repository URL (which would expose project names), the system takes the SHA256 hash and truncates to 16 hex characters. This is not anonymization — it is **pseudonymization**. An observer with access to the telemetry backend who knows a target repository's URL can trivially compute the hash and match it. The 16-character truncation provides a 64-bit collision space — effectively unique for realistic repository populations.

### 2.4 Tool Input Logging

By default, tool inputs are aggressively truncated:

```
// Source: metadata.ts (truncation thresholds)
Strings:       512 character hard cap, displayed as 128 + "…"
JSON objects:  4,096 character limit
Arrays:        maximum 20 items
Nested depth:  maximum 2 levels
```

But there is a **full-capture override**:

```typescript
// Source: metadata.ts (OTEL tool details)
// When OTEL_LOG_TOOL_DETAILS=1 is set in environment:
// ALL tool inputs are logged WITHOUT truncation
```

This environment variable, intended for debugging, creates a vector where every file path, every search query, and every code edit command is captured in full fidelity.

### 2.5 Bash Command Extension Tracking

A particularly granular collection mechanism targets Bash commands. When you run operations involving any of 17 commands (`rm`, `mv`, `cp`, `touch`, `mkdir`, `chmod`, `chown`, `cat`, `head`, `tail`, `sort`, `stat`, `diff`, `wc`, `grep`, `rg`, `sed`), the system extracts and logs the **file extensions** of the arguments. This creates a profile of what file types you work with — `.py`, `.ts`, `.go`, `.md` — without capturing specific file paths.

> → Cross-reference: [Episode 06: Bash Engine](06-bash-engine.md) for command parsing

---

## 3. The Opt-Out Dilemma

**Source coordinates**: `src/services/analytics/firstPartyEventLogger.ts:141-144`, `src/services/analytics/config.ts`

### 3.1 When Analytics Are Disabled

```typescript
// Source: src/services/analytics/config.ts
// isAnalyticsDisabled() returns true ONLY for:
// 1. Test environments (NODE_ENV !== 'production')
// 2. Third-party cloud providers (Bedrock, Vertex)
// 3. Global telemetry opt-out flag
```

The first-party logging function checks this gate:

```typescript
// Source: src/services/analytics/firstPartyEventLogger.ts:141-144
export function is1PEventLoggingEnabled(): boolean {
  return !isAnalyticsDisabled()
}
```

For direct Anthropic API users (the vast majority), `isAnalyticsDisabled()` returns `false`. There is **no settings panel, no CLI flag, and no environment variable** that a regular user can set to disable first-party event logging while maintaining full product functionality.

### 3.2 The Datadog Gate

The Datadog channel adds one more restriction:

```typescript
// Source: src/services/analytics/datadog.ts:168-171
// Don't send events for 3P providers (Bedrock, Vertex, Foundry)
if (getAPIProvider() !== 'firstParty') return
```

Third-party API providers (AWS Bedrock, Google Vertex, Foundry) are exempt from Datadog logging — because their billing and analytics flow through separate systems. But first-party users cannot escape.

### 3.3 The Sink Killswitch (Remote Off-Switch)

Ironically, Anthropic _can_ remotely disable analytics — but only for themselves:

```typescript
// Source: src/services/analytics/sinkKillswitch.ts
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'
// GrowthBook flag that can disable analytics sinks remotely
```

This GrowthBook flag allows Anthropic to globally or selectively silence the analytics pipeline. Individual users have no equivalent capability.

### 3.4 Regulatory Implications

The combination of: (1) no user-facing opt-out, (2) persistent device and session tracking, (3) repository fingerprinting, and (4) organization-level identification creates a dataset that falls squarely within the scope of GDPR Article 6 (lawful basis for processing) and CCPA Section 1798.100 (right to know). While the truncation and hashing measures reduce sensitivity, the persistent `deviceId` and `accountUUID` constitute identifiable data under most privacy frameworks.

> → Design pattern: The **stale-while-error** strategy on failed telemetry exports (disk-persist + retry) prioritizes delivery completeness over user control. This is a deliberate product decision that trades privacy transparency for operational observability.

---

## 4. Model Codename System

**Source coordinates**: `src/utils/undercover.ts:48-49`, `src/constants/prompts.ts`, `src/migrations/migrateFennecToOpus.ts`, `src/buddy/types.ts`

Anthropic assigns **animal codenames** to internal model versions — a practice common in tech companies, but Claude Code's source reveals the specific codenames, their evolutionary lineage, and the elaborate machinery built to prevent them from leaking.

### 4.1 The Four Known Codenames

| Codename | Animal | Role | Evidence |
|----------|--------|------|----------|
| **Capybara** | 水豚 | Sonnet-series model, currently v8 | `capybara-v2-fast[1m]` in model strings; dedicated prompt patches |
| **Tengu** | 天狗 | Product/telemetry prefix | All 250+ analytics events and feature flags use `tengu_*` prefix |
| **Fennec** | 耳廓狐 | Predecessor to Opus 4.6 | Migration script: `fennec-latest → opus` |
| **Numbat** | 袋食蚁兽 | Next unreleased model | Comment: `"Remove this section when we launch numbat"` |

### 4.2 The Evolution Chain

```
Fennec (耳廓狐)  ──migration──→  Opus 4.6  ──→  [Numbat?]
Capybara (水豚)  ──────────────→  Sonnet v8 ──→  [?]
Tengu (天狗)     ──────────────→  Product/telemetry prefix (not a model)
```

The Fennec-to-Opus migration is a concrete code artifact:

```typescript
// Source: src/migrations/migrateFennecToOpus.ts:7-11
// fennec-latest      → opus
// fennec-latest[1m]  → opus[1m]
// fennec-fast-latest → opus[1m] + fast mode enabled
```

### 4.3 Codename Protection Mechanisms

Two layers prevent codenames from leaking into external builds:

**Layer 1: Build-time scanner** — `scripts/excluded-strings.txt` contains patterns that the CI pipeline scans for in build output. Any match fails the build.

**Layer 2: Runtime obfuscation** — When a codename might appear in user-visible strings, it is actively masked:

```typescript
// Source: src/utils/model/model.ts:386-392
function maskModelCodename(baseName: string): string {
  // e.g. capybara-v2-fast → cap*****-v2-fast
  const [codename = '', ...rest] = baseName.split('-')
  const masked = codename.slice(0, 3) + '*'.repeat(Math.max(0, codename.length - 3))
  return [masked, ...rest].join('-')
}
```

**Layer 3: Source-level collision avoidance** — The Buddy virtual pet system includes "capybara" as a pet species, which collides with the model codename scanner. The solution is encoding the species name character-by-character at runtime to keep the literal out of the source bundle:

```typescript
// Source: src/buddy/types.ts:10-13
// One species name collides with a model-codename canary in excluded-strings.txt.
// The check greps build output (not source), so runtime-constructing the value
// keeps the literal out of the bundle while the check stays armed for the actual codename.
```

### 4.4 Capybara v8: Five Documented Behavioral Defects

The source code contains specific, annotated workarounds for Capybara v8 issues — a rare window into model-level debugging:

| # | Defect | Impact | Source Location |
|---|--------|--------|----------------|
| 1 | **Stop sequence false trigger** | ~10% rate when `<functions>` appears at prompt tail | `prompts.ts` / `messages.ts:2141` |
| 2 | **Empty tool_result zero output** | Model generates nothing when receiving blank tool results | `toolResultStorage.ts:281` |
| 3 | **Over-commenting** | Requires dedicated anti-commenting prompt patches | `prompts.ts:204` |
| 4 | **High false-claims rate** | 29-30% FC rate vs. Capybara v4's 16.7% | `prompts.ts:237` |
| 5 | **Insufficient verification** | Requires "thoroughness counterweight" prompt injection | `prompts.ts:210` |

Each defect has a `@[MODEL LAUNCH]` annotation tied to it, indicating these are temporary patches expected to be removed or revised when the next model (Numbat) launches. The codebase contains **8+ `@[MODEL LAUNCH]` markers** covering: default model names, family IDs, knowledge cutoff dates, pricing tables, context window configurations, thinking mode support, display name mappings, and migration scripts.

> → Cross-reference: [Episode 01: QueryEngine](01-query-engine.md) for how prompt patches affect the query loop

---

## 5. Feature Flag Obfuscation

**Source coordinates**: `src/services/analytics/growthbook.ts` (41KB), `src/services/analytics/sinkKillswitch.ts`, various `utils/*.ts`

### 5.1 The Tengu Naming Convention

Every feature flag and analytics event follows a deliberate naming pattern designed to obscure its purpose from external observers:

```
tengu_<word1>_<word2>
```

The word pairs are selected from a constrained vocabulary — adjective/material words paired with nature/object words — creating names that are memorable to insiders but opaque to outsiders. Examples from the actual codebase:

| Flag Name | Decoded Purpose | Category |
|-----------|----------------|----------|
| `tengu_frond_boric` | Analytics sink killswitch | Killswitch |
| `tengu_amber_quartz_disabled` | Voice mode emergency off | Killswitch |
| `tengu_amber_flint` | Agent teams gate | Feature gate |
| `tengu_hive_evidence` | Verification agent gate | Feature gate |
| `tengu_onyx_plover` | Auto-Dream (background memory) | Feature gate |
| `tengu_coral_fern` | Memdir feature | Feature gate |
| `tengu_moth_copse` | Memdir switch (secondary) | Feature gate |
| `tengu_herring_clock` | Team memory | Feature gate |
| `tengu_passport_quail` | Path feature | Feature gate |
| `tengu_slate_thimble` | Memdir switch (tertiary) | Feature gate |
| `tengu_sedge_lantern` | Away summary | Feature gate |
| `tengu_marble_sandcastle` | Fast mode (Penguin) gate | Feature gate |
| `tengu_penguins_off` | Fast mode disable | Killswitch |
| `tengu_turtle_carbon` | Ultrathink gate | Feature gate |
| `tengu_log_datadog_events` | Datadog event gate | Analytics |
| `tengu_event_sampling_config` | Per-event sampling rates | Config |
| `tengu_1p_event_batch_config` | 1P batch processor config | Config |
| `tengu_ant_model_override` | Internal model override | Internal |
| `tengu_max_version_config` | Version enforcement | Config |

### 5.2 The Three-Tier Flag Resolution

Feature behavior in Claude Code is controlled through three distinct mechanisms, each operating at a different level:

```
┌─────────────────────────────────────────────────────────────────┐
│  Tier 1: Compile-Time DCE (Dead Code Elimination)               │
│  mechanism: feature('FLAG_NAME') from bun:bundle                │
│  scope:     entire code branches removed at build time          │
│  examples:  VOICE_MODE, DAEMON, KAIROS, COORDINATOR_MODE        │
│  effect:    code doesn't exist in published bundle at all       │
├─────────────────────────────────────────────────────────────────┤
│  Tier 2: Runtime Environment Check                              │
│  mechanism: process.env.USER_TYPE === 'ant'                     │
│  scope:     code exists but is bypassed for external users      │
│  examples:  REPLTool, TungstenTool, debug logging               │
│  effect:    constant-folded by V8 JIT after first check         │
├─────────────────────────────────────────────────────────────────┤
│  Tier 3: Runtime GrowthBook Flag                                │
│  mechanism: getFeatureValue('tengu_*') via GrowthBook SDK       │
│  scope:     can change per-user, per-session, per-experiment    │
│  examples:  all tengu_* flags listed above                      │
│  effect:    cached locally, refreshed periodically              │
└─────────────────────────────────────────────────────────────────┘
```

Some features use **double-gating** — a Tier 1 compile-time gate combined with a Tier 3 runtime flag:

```typescript
// Source: src/utils/thinking.ts
export function isUltrathinkEnabled(): boolean {
  if (!feature('ULTRATHINK')) return false      // Tier 1: DCE in external builds
  return getFeatureValue_CACHED_MAY_BE_STALE(   // Tier 3: runtime toggle
    'tengu_turtle_carbon', true
  )
}
```

In external builds, `feature('ULTRATHINK')` is `false`, so the entire function body — including the GrowthBook call — is dead-code-eliminated. In internal builds, the runtime flag provides dynamic control. This two-tier approach means Anthropic can both restrict features to internal builds AND dynamically control them within those builds.

### 5.3 The Sink Killswitch Architecture

The analytics killswitch (`tengu_frond_boric`) deserves special attention for its engineering subtlety:

```typescript
// Source: src/services/analytics/sinkKillswitch.ts
export function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<
    Partial<Record<SinkName, boolean>>
  >(SINK_KILLSWITCH_CONFIG_NAME, {})
  // NOTE: Must NOT be called from is1PEventLoggingEnabled() —
  // growthbook.ts:isGrowthBookEnabled() calls that, creating a cycle
  return config?.[sink] === true
}
```

The comment reveals a **circular dependency hazard**: GrowthBook initialization calls `is1PEventLoggingEnabled()`, which would call `isSinkKilled()`, which calls GrowthBook — infinite recursion. The solution: the killswitch is checked at **each event dispatch site**, not at the global enable check. This is a textbook example of breaking circular dependencies through architectural placement rather than lazy loading.

> → Cross-reference: [Episode 16: Infrastructure](16-infrastructure-config.md) §1.3 for the latching pattern used in analytics beta headers

---
