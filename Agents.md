# How OpenVault Memory Actually Works

A miniature cognitive architecture inspired by the [Generative Agents (Smallville) paper](https://arxiv.org/abs/2304.03442).
All state lives in `context.chatMetadata.openvault` — no external database.

---

## 1. Extraction — How Memories Get Created

A background worker fires on every new message (fire-and-forget, singleton via `isWorkerRunning()`).
It runs a 6-stage pipeline:

| Stage | Function | What It Does |
|-------|----------|--------------|
| 1 | `fetchEventsFromLLM` | Sends recent chat to the LLM, extracts factual **events** with importance (1-5), characters involved, witnesses, emotional impact, locations |
| 2 | `fetchGraphFromLLM` | Extracts **entities** (PERSON, PLACE, ORG, OBJECT, CONCEPT) and **relationships**. Updates a knowledge graph with semantic merge |
| 3 | `enrichAndDedupEvents` | Removes duplicates via cosine similarity (>=95%) + Jaccard token overlap. Surviving events get a `mentions` counter bump on dedup hits |
| 4 | `processGraphUpdates` | Consolidates edge descriptions, runs **Louvain community detection** to cluster related entities |
| 5 | `synthesizeReflections` | Deferred on backfill — see Reflection section below |
| 6 | `synthesizeCommunities` | Deferred on backfill — summarizes entity clusters |

**Batch alignment**: Messages are processed in batches aligned to turn boundaries (Bot-to-User transitions).

**Swipe protection**: `trimTailTurns()` strips the last N complete turns to avoid extracting half-swiped messages.

**Emergency Cut**: `executeEmergencyCut()` extracts everything, then hides original messages from the LLM context (`is_system=true`). The LLM only sees: preset, char card, lorebooks, and OpenVault memories.

**Key files**: `src/extraction/extract.js`, `src/extraction/worker.js`, `src/extraction/scheduler.js`, `src/extraction/structured.js`

---

## 2. Storage — Where Memories Live

Everything is a JSON blob in SillyTavern's chat metadata. Key structures:

### `memories[]` — Events AND reflections
```
{
  id: string,
  type: "event" | "reflection",
  summary: string,                          // The actual memory text
  importance: 1 | 2 | 3 | 4 | 5,           // LLM-rated significance
  tokens: string[],                         // Pre-computed BM25 stems
  message_ids?: number[],                   // Source chat messages (events)
  source_ids?: string[],                    // Cited evidence (reflections)
  level?: number,                           // Reflection hierarchy depth (1 = from events, 2+ = from reflections)
  parent_ids?: string[],                    // Ancestor reflection IDs (level 2+)
  temporal_anchor: string | null,           // Extracted timestamp ("Friday, 3:40 PM")
  is_transient: boolean,                    // Short-term intentions decay 5x faster
  characters_involved: string[],
  witnesses: string[],
  embedding_b64: string,                    // Base64 Float32Array
  archived: boolean,                        // Soft-deleted, ignored by retrieval
  mentions?: number,                        // Frequency boost (increments on dedup overlap)
  retrieval_hits?: number                   // Hit-damping counter (frequently recalled = slower fade)
}
```

### `graph{}`
- **nodes**: Entity graph keyed by normalized name (Person/Place/Org/Object/Concept) with embeddings
- **edges**: Relationships between entities with weight scores, keyed `sourceKey__targetKey`
- Semantic merge: if "Gwen" and "Gwen Stacy" are close enough in embedding space, they merge with redirect chains

### `communities{}`
- Louvain-detected clusters of related entities with summaries
- Regenerated every 100 messages

### Supporting structures
- `global_world_state` — High-level world summary
- `character_states{}` — Per-character: current emotion, intensity, known_events (POV strictness)
- `reflection_state{}` — Per-character importance accumulator (triggers reflection at >=40)
- `idf_cache{}` — Pre-computed BM25 inverse document frequency map
- `processed_message_ids[]` — Tracks which messages have been extracted

Memories can optionally be synced to SillyTavern's ST Vector storage (server-side vector search via REST API).

**Key files**: `src/store/chat-data.js`, `src/store/schemas.js`, `include/DATA_SCHEMA.md`

---

## 3. Retrieval — How Memories Get Back Into the Prompt

Hybrid alpha-blend scoring system. Final formula:

```
Score = (Base + (Alpha * VectorBonus) + ((1 - Alpha) * BM25Bonus)) * FrequencyFactor
```

### Base Score (Forgetfulness Curve)
```
Base = Importance * e^(-lambda * distance)
lambda = (baseLambda / importance^2) * hitDamping
hitDamping = max(0.5, 1 / (1 + retrieval_hits * 0.1))
```

- Importance 5 decays 25x slower than importance 1
- Importance 5 has a **soft floor of 1.0** — never fully disappears
- Frequently retrieved memories decay up to 50% slower (hit-damping)
- Transient memories get `lambda *= 5.0` (5x faster fade)
- Higher-level reflections decay 2x slower per level (max level 3)

### Vector Similarity Bonus (Alpha-Blend)
- Cosine similarity between query embedding and memory embedding
- Only computed above a threshold (default 0.5)
- Weighted by `alpha` (default 0.7 — favors vector similarity over BM25)

### 4-Tier BM25 Keyword Matching
IDF cached at extraction time. POV names stripped as stopwords.

| Layer | Content | Boost |
|-------|---------|-------|
| 0 | Exact multi-word entity phrases | 10x maxIDF |
| 1 | Single-word graph entities (stemmed) | 5x |
| 2 | Corpus-grounded user-message stems | 3x |
| 3 | Non-grounded user-message stems | 2x |

### Two-Pass Optimization
1. **Fast Pass**: Score all memories with Base + BM25 (O(N) cheap math)
2. **Cutoff**: Take top 200 candidates (`VECTOR_PASS_LIMIT`)
3. **Slow Pass**: Compute expensive cosine similarity only on those 200

### Context Budgeting (Score-First Soft Balancing)
- Phase 1: Reserve 20% per chronological bucket (Old / Mid / Recent), fill with highest-scoring from each
- Phase 2: Remaining 40% allocated purely by score regardless of bucket

### Injection
Memories get injected via SillyTavern's extension prompt system at configurable position/depth. Available as `{{openvault::memory}}` and `{{openvault::world}}` macros.

**Key files**: `src/retrieval/scoring.js`, `src/retrieval/retrieve.js`, `src/retrieval/math.js`, `src/retrieval/query-context.js`, `src/retrieval/formatting.js`

---

## 4. Forgetting — Does It Forget Things?

**Yes — but it's soft forgetting (lowered score), not deletion.**

| Mechanism | What Happens | Scope |
|-----------|-------------|-------|
| Exponential decay | Score drops over distance (messages) | All memories |
| Hit-damping | Frequently recalled memories decay up to 50% slower | All memories |
| Transient flag | Short-term intentions fade 5x faster | Events flagged `is_transient: true` |
| Importance floor | Importance-5 memories never drop below score 1.0 | Importance 5 only |
| Level-aware decay | Higher-level reflections decay 2x slower per level | Reflections level 2+ |
| Archiving | Soft-deleted — completely ignored by retrieval, scoring, IDF | Reflections only |

**What's never automatically deleted**: Raw events (`type: "event"`) accumulate forever. They only get archived if manually deleted. Only reflections get archived automatically.

**Manual deletion**: Users can delete individual memories from the UI. `deleteMemory(id)` in `chat-data.js`.

**Key files**: `src/retrieval/math.js` (scoring), `src/reflection/reflect.js` (archiving)

---

## 5. Reflection — "Thinking About Thinking"

Per-character system inspired by Smallville. Synthesizes raw events into high-level psychological insights.

### Pipeline
1. **Accumulate**: Each extracted event adds its `importance` to involved characters' `importance_sum`
2. **Trigger**: When `importance_sum >= 40` for a character, reflection begins
3. **Pre-flight gate**: If top recent events are >85% similar to existing reflections, skip (saves LLM tokens)
4. **Candidate set**: Top 50 recent events + all old reflections (enables level 2+ meta-synthesis)
5. **POV filter**: Character can only reflect on things they witnessed (`filterMemoriesByPOV()`)
6. **Generate**: Single unified LLM call produces 1-3 question+insight pairs with evidence citations
7. **3-Tier dedup**:
   - >= 90% similarity to existing: **Reject** (concept already exists)
   - 80-89%: **Replace** (same theme, evidence evolved — old one archived)
   - < 80%: **Add** (genuinely new insight)
8. **Reset**: Importance accumulator cleared after successful reflection

### Reflection Properties
- `type: "reflection"` (distinguished from events)
- `importance: 4` (fixed default — high but not max)
- `level`: Hierarchy depth (1 = from events, 2+ = synthesized from other reflections)
- `parent_ids`: Source reflection IDs for level 2+
- `source_ids`: Cited evidence memory IDs
- `witnesses`: Only the reflecting character (internal thought)

### Budget Controls
- Hard cap: 50 reflections per character (oldest archived when exceeded)
- Pre-flight gate: Skips generation when recent events align with existing insights
- Max level: 3 (prevents runaway abstraction)

**Key files**: `src/reflection/reflect.js`, `src/prompts/reflection/`

---

## 6. Entity Graph

Separate from the memory stream. A knowledge graph that grows over time.

- **Semantic merge**: If the LLM extracts "Gwen" and later "Gwen Stacy", the system checks embedding similarity and merges if close enough. Maintains `_mergeRedirects` for old key resolution.
- **Edge consolidation**: When edges accumulate enough, an LLM call summarizes the relationship description.
- **Louvain community detection**: Groups related entities into clusters. Runs every 100 messages.
- **Hairball prevention**: Main character edges temporarily pruned during community detection to avoid them dominating the graph.
- **ST Vector sync**: Nodes and communities can be synced to server-side vector storage for faster retrieval.

**Key files**: `src/graph/graph.js`, `src/graph/communities.js`

---

## 7. Architecture Summary

```
Chat Messages
    |
    v
[Background Worker] ──fires on every message──>
    |
    v
[Extraction Pipeline]
    |-- Stage 1: Event Extraction (LLM)     --> memories[] (type: "event")
    |-- Stage 2: Graph Extraction (LLM)     --> graph.nodes / graph.edges
    |-- Stage 3: Dedup & Enrich            --> mentions++, embeddings computed
    |-- Stage 4: Graph Consolidation        --> edges summarized, communities detected
    |-- Stage 5: Reflection (LLM)           --> memories[] (type: "reflection")
    |-- Stage 6: Community Summary (LLM)    --> communities{}
    |
    v
[Storage: chatMetadata.openvault]
    |
    v
[On Each User Message → Retrieval]
    |-- Build query from last 3 user messages + entities
    |-- Score all memories: (Base + Vector + BM25) * Frequency
    |-- Budget: 20% per time bucket + 40% score-first
    |-- Inject into prompt at configured position/depth
    |
    v
[LLM sees: preset + char card + lorebooks + OpenVault memories]
```

---

## Key Constants (src/constants.js)

| Setting | Default | Effect |
|---------|---------|--------|
| `forgetfulnessBaseLambda` | 0.05 | Base exponential decay rate |
| `transientDecayMultiplier` | 5.0 | Short-term memory fade speedup |
| `alpha` | 0.7 | Vector vs BM25 blend (1.0 = vector only) |
| `vectorSimilarityThreshold` | 0.5 | Min similarity for vector bonus |
| `reflectionThreshold` | 40 | Importance sum to trigger reflection |
| `maxReflectionsPerCharacter` | 50 | Hard cap before archiving |
| `maxReflectionLevel` | 3 | Max reflection hierarchy depth |
| `reflectionLevelMultiplier` | 2.0 | Decay slowdown per reflection level |
| `dedupSimilarityThreshold` | 0.95 | Cosine similarity to reject duplicate events |
| `communityDetectionInterval` | 100 | Messages between community regeneration |
