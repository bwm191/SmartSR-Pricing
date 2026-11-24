# SmartSR-Pricing

English | [简体中文](README.zh-CN.md)

SmartSR-Pricing is the public, versioned pricing registry used by [SmartSR-Core](https://github.com/bwm191/SmartSR-Core) to estimate LLM token costs. It stores provider/model/endpoint pricing records, source references, aliases, and version metadata.

This repository contains pricing metadata only. It does not make API calls, store credentials, or determine which models a study must use.

## Quick access

- **Latest version index:** [latest.json](latest.json)
- **Current snapshot:** [v2026-11-24.json](v2026-11-24.json)
- **Raw index URL:** https://raw.githubusercontent.com/bwm191/SmartSR-Pricing/main/latest.json
- **Raw current snapshot URL:** https://raw.githubusercontent.com/bwm191/SmartSR-Pricing/main/v2026-11-24.json

The index is the entry point for consumers. Do not assume that the newest snapshot will always have the same filename.

## How the registry works

The repository uses two levels of JSON files:

1. `latest.json` is a small index. It identifies the latest published version and maps each version to a registry file and its SHA-256 digest.
2. `vYYYY-MM-DD.json` is a versioned pricing snapshot. Treat published snapshots as immutable: publish a new file when pricing changes rather than silently editing an existing version.

A consumer should:

1. Fetch `latest.json`.
2. Read `latest_version`.
3. Find that version in `versions` and obtain its `path` and `sha256`.
4. Fetch the referenced registry file.
5. Verify the SHA-256 digest against the raw file bytes before using the data.
6. Validate the registry schema and resolve the configured provider, model, and endpoint.

SmartSR-Core follows this index-and-snapshot approach and records the resolved version and provenance in its pricing snapshot.

## Registry structure

A registry snapshot has the following top-level fields:

| Field | Meaning |
| --- | --- |
| `schema_version` | Registry schema identifier; currently `smartsr.pricing-registry.v1`. |
| `registry_version` | Version identifier; must match the version selected from the index. |
| `published_at` | Publication timestamp in ISO 8601 format. |
| `currency` | Currency used by all rates in this snapshot, e.g. `USD`. |
| `models` | List of provider/model/endpoint pricing records. |

Each model record includes:

| Field | Meaning |
| --- | --- |
| `provider` | Provider key, e.g. `deepseek` or `qwen`. |
| `model` | Canonical model identifier used for matching. |
| `aliases` | Alternative model identifiers accepted by the resolver. |
| `base_url` | API endpoint. It is part of the match key; use the endpoint configured in the project. |
| `deployment_scope` | Region or deployment scope to which the price applies. |
| `input_per_1m` | Input-token price per one million tokens, in the snapshot currency. |
| `output_per_1m` | Output-token price per one million tokens, in the snapshot currency. |
| `input_basis` | Short description of the pricing basis used for the top-level input rate. |
| `source_url` | Provider pricing documentation or other source used for verification. |
| `checked_at` | Date the source was last checked. |

Optional fields may preserve additional provider-specific information, such as cache-hit rates, peak/off-peak rates, context-length tiers, model-version notes, or pricing caveats. These fields provide provenance and context; they are not necessarily interpreted by every SmartSR-Core version.

### Units and matching

- All top-level rates are expressed **per 1 million tokens**, not per 1,000 tokens or per token.
- The registry currency applies to every model record in the same snapshot.
- SmartSR-Core converts the top-level rates to per-1,000-token rates for cost calculation.
- Matching uses the provider, model, and API endpoint together. The provider is compared case-insensitively and a trailing slash is removed from the endpoint. The canonical model identifier is checked first; aliases are checked only if no canonical match is found.
- Model identifiers and aliases should match the actual configured values exactly. A correct model name with a different endpoint will not resolve.

The registry is role-independent. TA roles such as `LLM_A` and `LLM_B`, or other workflow roles, are mapped to provider/model/endpoint metadata by SmartSR-Core. A model record is not an instruction to use that model in a particular study.

## Current snapshot: 2026-11-24

The current index points to `v2026-11-24.json`. Its currency is USD.

| Registry entry | Endpoint | Top-level input rate | Top-level output rate | Basis |
| --- | --- | ---: | ---: | --- |
| `deepseek/deepseek-flash` (alias: `deepseek-v4-flash`) | `https://api.deepseek.com` or `https://api.deepseek.com/v1` | $0.30 / 1M | $1.20 / 1M | Peak-period cache-miss rate; conservative flat-rate basis |
| `qwen/qwen3.7-flash` | `https://dashscope.aliyuncs.com/compatible-mode/v1` | $0.028 / 1M | $0.11 / 1M | Standard real-time, 0–32K input tier |

The snapshot also retains additional provider-specific details:

- **DeepSeek:** cache-hit and off-peak rates are recorded in `pricing_details`. The top-level rates represent the peak-period cache-miss basis.
- **Qwen3.7 Flash:** additional context-length tiers are recorded in `tiered_rates_per_1m`: up to 256K input tokens and up to 1M input tokens.

### Important calculation limitation

The current SmartSR-Core v0.3.1 resolver uses only `input_per_1m` and `output_per_1m` as a single flat rate for each model. It does **not** yet calculate costs dynamically by cache-hit status, peak/off-peak period, or input-length tier.

Consequently:

- DeepSeek costs are estimated using the conservative top-level peak/cache-miss rates, not the actual mix of cache hits and off-peak requests.
- Qwen requests above 32K input tokens are still estimated using the 0–32K top-level rate. The estimate may therefore be lower than the provider's tiered charge.

The extra fields preserve the information needed for a future pricing-engine upgrade; their presence must not be mistaken for support in the current calculator.

## Using a version with SmartSR-Core

By default, SmartSR-Core can resolve the latest published registry through the index. To pin a project to a specific snapshot, set:

```yaml
cost_tracking:
  pricing_version: "2026-11-24"
```

The exact project configuration may contain other `cost_tracking` settings; add this field without replacing unrelated settings.

Important:

- An explicit `cost_tracking.rates` configuration takes precedence over the public registry.
- A pinned `pricing_version` selects that immutable snapshot rather than following the latest index.
- If a provider/model/endpoint combination is not found, the resolver reports that pricing is not configured. It does not invent a fallback price.
- The registry does not contain API keys or other secrets. Credentials must be managed separately through the provider's supported secret or environment-variable mechanism.

See the [SmartSR-Core README](https://github.com/bwm191/SmartSR-Core) for workflow and configuration details.

## Publishing a new pricing version

Create a new snapshot instead of editing an already published version.

1. Check the provider's current pricing documentation. Record the source URL, date checked, currency, deployment region, model identifier, endpoint, and pricing basis.
2. Add a new file named `vYYYY-MM-DD.json`. Set `registry_version` to the same date and preserve the required schema fields.
3. Validate the JSON and check that each provider/model/endpoint combination is unique.
4. Calculate the SHA-256 digest from the exact bytes of the completed snapshot file.
5. Update `latest.json`: set `latest_version` to the new version and add its relative path and SHA-256 digest under `versions`.
6. Commit the new snapshot and index together.

Example digest calculation:

```python
from hashlib import sha256
from pathlib import Path

registry_path = Path("vYYYY-MM-DD.json")
digest = sha256(registry_path.read_bytes()).hexdigest()
print(digest)
```

The digest must be calculated after the file is finalized. Any subsequent byte-level change to that snapshot requires a new digest and a corresponding index update. For auditability, prefer publishing a new version instead.

## Validation checklist

Before publishing, confirm that:

- [ ] The JSON is valid UTF-8 and follows the registry schema.
- [ ] `registry_version` matches the version key in `latest.json`.
- [ ] Every model record contains all required fields.
- [ ] Rates are finite, non-negative numbers expressed per one million tokens.
- [ ] The currency is correct and consistent across the snapshot.
- [ ] The provider, canonical model ID, aliases, and `base_url` match the intended API deployment.
- [ ] The source URL supports the recorded price and pricing basis.
- [ ] `checked_at` reflects the actual verification date.
- [ ] The SHA-256 in `latest.json` matches the exact snapshot bytes.
- [ ] Existing versioned snapshots have not been silently modified.

## Scope and disclaimer

This repository is a versioned reference for software-based cost estimation. Provider prices, discounts, cache policies, regional availability, and billing rules can change. The registry records the source and verification date for each entry, but it is not a billing system and does not guarantee that an estimate will equal a provider invoice. Always check the linked provider documentation before making a material cost decision.
