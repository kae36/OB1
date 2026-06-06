# OpenRouter Endpoint Audit — OB1 Integration Assessment

**Date:** 2026-05-05  
**Status:** OpenRouter removing Supabase sub-processor (does NOT directly affect our code)  
**Risk Level:** LOW — Our code uses OpenRouter's public `/embeddings` and `/chat/completions` endpoints, not Supabase

---

## Executive Summary

OB1 uses OpenRouter as a managed API gateway to access embedding and LLM models. We make HTTP calls to **two public endpoints** that should remain stable:

- **`POST /embeddings`** — Generate vector embeddings (critical path)
- **`POST /chat/completions`** — Extract metadata and classifications

OpenRouter's discontinuation of Supabase as a sub-processor is an **internal infrastructure change** and does not affect the public API surface our code depends on. However, we should monitor for:
- API deprecations or breaking changes in OpenRouter's `/embeddings` endpoint
- Model availability changes or routing adjustments
- Rate limits or quota changes

---

## Endpoint Call Sites

### 1. `/embeddings` — CRITICAL PATH (12+ call sites)

**Endpoint:** `POST https://openrouter.ai/api/v1/embeddings`  
**Authentication:** Bearer token (OPENROUTER_API_KEY)  
**Models Used:**
- `openai/text-embedding-3-small` (primary, configurable)

**Request Contract:**
```json
{
  "model": "openai/text-embedding-3-small",
  "input": "text to embed"
}
```

**Response Contract:**
```json
{
  "data": [
    {
      "embedding": [0.123, 0.456, ...]
    }
  ]
}
```

**Call Sites:**

| Location | Function | Purpose | Criticality |
|----------|----------|---------|-------------|
| `server/index.ts:18` | `getEmbedding()` | MCP server embedding endpoint | **CRITICAL** |
| `recipes/repo-learning-coach/server/brain.ts:58` | `getEmbedding()` | Learning coach artifact capture | **CRITICAL** |
| `recipes/grok-export-import/import-grok.mjs:89` | (inline) | Grok export ingestion | Medium |
| `recipes/journals-blogger-import/import-blogger.mjs:123` | (inline) | Blogger import | Medium |
| `recipes/x-twitter-import/import-x-twitter.mjs:79` | (inline) | X/Twitter import | Medium |
| `recipes/instagram-import/import-instagram.mjs:82` | (inline) | Instagram import | Medium |
| `recipes/email-history-import/pull-gmail.ts` | `getEmbedding()` | Gmail history import | Medium |
| `recipes/google-activity-import/import-google-activity.mjs:29` | (inline) | Google Activity import | Medium |
| `integrations/kubernetes-deployment/index.ts:130` | `qEmb` | K8s deployment integration | Medium |
| `integrations/entity-extraction-worker/_shared/helpers.ts:105` | (inline) | Entity extraction | Medium |
| `recipes/obsidian-vault-import/import-obsidian.py:326` | (inline) | Obsidian import (Python) | Medium |
| `recipes/perplexity-conversation-import/import-perplexity.py` | (inline) | Perplexity import (Python) | Medium |

**Error Handling:**
- All implementations check `response.ok` and throw on HTTP errors ✓
- All expect `data[0].embedding` to exist, throw if missing ✓
- Common error responses handled: auth failures, rate limits, malformed requests ✓

---

### 2. `/chat/completions` — METADATA EXTRACTION (7 call sites)

**Endpoint:** `POST https://openrouter.ai/api/v1/chat/completions`  
**Authentication:** Bearer token (OPENROUTER_API_KEY)  
**Models Used:**
- `openai/gpt-4o-mini` (primary)
- `openai/gpt-4o` (alternate)

**Request Contract (example):**
```json
{
  "model": "openai/gpt-4o-mini",
  "response_format": { "type": "json_object" },
  "messages": [
    { "role": "system", "content": "Extract metadata..." },
    { "role": "user", "content": "user input" }
  ]
}
```

**Response Contract:**
```json
{
  "choices": [
    {
      "message": {
        "content": "{\"people\": [], \"topics\": [...]}"
      }
    }
  ]
}
```

**Call Sites:**

| Location | Purpose | Models | Error Fallback |
|----------|---------|--------|----------------|
| `server/index.ts:38` | Metadata extraction (MCP) | gpt-4o-mini | Returns generic JSON ✓ |
| `recipes/source-filtering/backfill-metadata.ts:68` | Backfill metadata | gpt-4o-mini | Throws error (batch script) |
| `recipes/adaptive-capture-classification/capture-with-gating.ts:104` | Capture gating | gpt-4o-mini | Not specified |
| `recipes/wiki-synthesis/scripts/synthesize-wiki.mjs:191` | Wiki synthesis | configurable (LLM_BASE_URL) | Not specified |
| `recipes/wiki-synthesis/scripts/backfill-gmail-wikis.mjs:264` | Wiki backfill | configurable (LLM_BASE_URL) | Not specified |
| `recipes/thought-enrichment/enrich-thoughts.mjs:138` | Thought enrichment | LLM_BASE_URL or gpt-4o-mini | Not specified |
| `recipes/entity-wiki/generate-wiki.mjs:517` | Entity wiki generation | configurable (LLM_BASE_URL) | Not specified |
| `recipes/obsidian-vault-import/import-obsidian.py:287` | Obsidian import (Python) | gpt-4o-mini | Not specified |
| `integrations/entity-extraction-worker/_shared/helpers.ts:171` | Entity extraction | gpt-4o-mini | Not specified |
| `integrations/entity-extraction-worker/index.ts:302` | Entity extraction | gpt-4o-mini | Wrapped in timeout |

---

## Configuration & Environment Variables

### Required
- **OPENROUTER_API_KEY** — Bearer token from https://openrouter.ai/keys
- **OPENROUTER_EMBEDDING_MODEL** — (optional, defaults to `openai/text-embedding-3-small`)

### Optional
- **LLM_BASE_URL** — Used by wiki synthesis recipes (defaults to OpenRouter)
- **EMBEDDING_API_BASE** — Used by K8s deployment integration
- **EMBEDDING_MODEL** — Custom model name for embeddings

### Discovered in Code
- `OPENROUTER_EMBEDDING_MODEL` → `recipes/repo-learning-coach/server/supabase.ts:17`
- `LLM_BASE_URL` → Wiki synthesis, entity wiki (allows non-OpenRouter providers)

---

## Potential Breaking Changes & Mitigation

### Risk 1: OpenRouter Discontinues `/embeddings` Endpoint
**Likelihood:** LOW (core service)  
**Impact:** All embedding ingestion fails  
**Mitigation:**
- Add fallback to direct OpenAI API
- Support swappable embedding providers via `EMBEDDING_API_BASE`
- Document migration path in README

### Risk 2: Model Deprecation (`text-embedding-3-small`, `gpt-4o-mini`)
**Likelihood:** MEDIUM (yearly)  
**Impact:** Calls return 404 or routing errors  
**Mitigation:**
- Monitor OpenRouter `/models` endpoint for discontinuations
- Make model names environment-configurable (already done for LLM_BASE_URL)
- Create migration guide for switching models

### Risk 3: Rate Limits or Quota Changes
**Likelihood:** MEDIUM (happens with OpenRouter updates)  
**Impact:** Batch import scripts fail; MCP endpoints return 429  
**Mitigation:**
- Add exponential backoff retry logic (not currently implemented)
- Document recommended batch sizes and rate limits
- Add telemetry to track quota exhaustion

### Risk 4: Response Format Changes
**Likelihood:** LOW (OpenAI-compatible, stable)  
**Impact:** JSON parsing fails, fallback metadata used  
**Mitigation:**
- Already implemented: try/catch JSON parse with fallback ✓
- Validate response shape before accessing `.data[0].embedding`

---

## Current Error Handling Assessment

### Strong ✓
- `server/index.ts`: HTTP status checks, embedding existence check ✓
- `brain.ts`: Same robustness ✓
- `backfill-metadata.ts`: HTTP checks, try/catch JSON parsing ✓

### Weak ⚠️
- No retry logic on transient failures (429, 5xx)
- No timeout enforcement (could hang indefinitely)
- Some scripts don't validate response structure before access
- No telemetry for quota exhaustion or rate limiting

---

## Recommendations

### Immediate (No action required, safe as-is)
✓ Code uses stable, public OpenRouter endpoints  
✓ Error handling is adequate for current use  
✓ Supabase removal doesn't affect our API calls  

### Short-term (Next release)
1. **Add retry logic** with exponential backoff for transient failures
   - Target: `server/index.ts` and `brain.ts` embedding calls
   - Implement: 3 retries with 100ms→1s→5s delays

2. **Document model availability**
   - Add `docs/OPENROUTER_MODELS.md` listing used models and deprecation history
   - Link from relevant READMEs

3. **Make embedding model configurable** in all recipes
   - Currently hardcoded in several import scripts
   - Use env var: `OPENROUTER_EMBEDDING_MODEL` (already done in repo-learning-coach)

### Medium-term (Before model deprecations)
4. **Support alternative embedding providers**
   - Add conditional logic: use EMBEDDING_API_BASE if provided, fallback to OpenRouter
   - Already partially done in K8s integration

5. **Add rate limiting awareness**
   - Check OpenRouter response headers for quota info
   - Document batch size limits per recipe

---

## Testing Checklist

To verify changes to OpenRouter integration:

- [ ] Test `/embeddings` endpoint with various input lengths (empty, 8k+ tokens)
- [ ] Test `/chat/completions` with JSON response format
- [ ] Simulate 429 (rate limit) and 5xx errors, verify retry behavior
- [ ] Verify error messages are user-facing and actionable
- [ ] Test with missing `OPENROUTER_API_KEY` env var
- [ ] Monitor API response times for latency regression

---

## Files to Watch

```
server/index.ts                                      # Core MCP endpoint
recipes/repo-learning-coach/server/brain.ts         # Learning coach
recipes/source-filtering/backfill-metadata.ts       # Backfill utility
recipes/*/import-*.mjs                              # Batch imports
integrations/entity-extraction-worker/              # Entity extraction
```

Monitor these files for OpenRouter calls when implementing fallbacks or retry logic.

---

## Related Resources

- OpenRouter API Docs: https://openrouter.ai/docs
- OpenRouter Models Endpoint: `curl https://openrouter.ai/api/v1/models`
- OB1 CLAUDE.md: Guard Rails & Guard Rails section
