# Decision models for outfit scoring and semantic search

Research for issue #7 (map #2). Researched 2026-11-10 against vendor and platform documentation. Terms follow `GLOSSARY.md` (Piece, Wardrobe, Outfit, Wear, ...). Everything below is the docs' claim as of that date, unless marked "unverified" or "inference".

## Question

What can decision models do for Vestis, and which provider should we use? Primary candidate: Jev by TypeSafe AI. Compare alternatives for (a) scoring and ranking Outfits and (b) embeddings for semantic search over a Wardrobe.

## Short answer

- **(a) Outfit scoring: use Jev (or any decision model) behind a thin port in the backend.** Generate candidate Outfits in code (graph-based generation), then ask Jev narrow Score / Noul questions per candidate and combine the scores with weights in code (TypeSafe's own "composite scoring" pattern). Cost is negligible (about $0.00002 per scored Outfit). Quality on fashion judgment is unproven and needs our own eval set.
- **(b) Semantic search: Jev is the wrong tool.** It returns no vectors. Use `gemini-embedding-2` (text and images in one shared space) called from the backend, stored in Firestore vector search. Jev can optionally re-rank a shortlist.
- **Gemini via Firebase AI Logic is not a fit for (b)**: its docs state embeddings and semantic retrieval are not supported there. It is a reasonable fallback or alternative for (a), and the only option here that reads photos in production.
- **Biggest risks:** Jev is "early access" with an experimental Gateway API and a preview-only image mode; its image support is contradictory across Vercel and TypeSafe pages; the `jev-latest` alias moves.

## 1. Jev (TypeSafe AI)

### What it is
"System One" decision model: input is `state` (text, JSON or images) plus named typed questions; output is typed answers with probabilities, no generated text. All questions in a request are answered in parallel against the same state. Source: [TypeSafe docs intro](https://docs.typesafe.ai/introduction.md), [System One concept](https://docs.typesafe.ai/concepts/system-one.md), [primitives index](https://docs.typesafe.ai/llms.txt).

Question types: Choice (one of N options, per-option probabilities, confidence), Score (ordered rubric levels, interpolated score, per-level probabilities, confidence), Noul (probability that a yes/no is true), Box (locate in image, 1.14 only). Gateway docs name Noul `boolean`/`predicate`.

### Access and status
- Homepage: "Try our first System One Model, Jev, in early access"; no waitlist mentioned; contact page for questions. ([typesafe.ai](https://typesafe.ai))
- API keys from the TypeSafe console (`console.typesafe.ai/keys`); base URL `https://api.typesafe.ai`, `POST /v1/systemone`, Bearer auth. ([quickstart](https://docs.typesafe.ai/introduction/quickstart.md))
- Launched 2026-09-15 per the brief; Jev 1.14 published 2026-10-14 ([model changelog](https://docs.typesafe.ai/changelog.md)). The changelog lists only 1.14.
- Official Python and JavaScript SDKs exist ([SDK index](https://docs.typesafe.ai/llms.txt)). No Dart/Flutter SDK; HTTP from the backend is the integration (fits ADR-0001 anyway).

### Pricing
- $42 per billion input tokens ($0.042 per Mtok), output tokens free, both 1.13 and 1.14 ([models](https://docs.typesafe.ai/models.md)). Same figure on the [Vercel model page](https://vercel.com/ai-gateway/models/jev.md) and [Eden AI docs](https://www.edenai.co/docs/v3/llms/decisions.md).
- Vercel's example response: 282 input tokens cost $0.000011844 ([Vercel Decision docs](https://vercel.com/docs/ai-gateway/modalities/decision)), consistent with $0.042/Mtok and no gateway surcharge in that example.
- Inference: an Outfit of 4-6 Pieces as compact JSON plus 5-10 questions is roughly 500-1,500 input tokens, so about $0.00002-0.00006 per Outfit; 100k scorings is about $2-6.
- Not stated anywhere I found: whether the price is promotional.

### Latency
TypeSafe's homepage shows one example workflow at 0.114 s vs 8.6 s for LLMs and claims 193.6x faster / 444.6x cheaper on its own workflows ([typesafe.ai](https://typesafe.ai)). The System One docs give no latency figures. Treat as unverified; measure ourselves. Vercel's own changelog also flags these as TypeSafe's claims ([Vercel changelog, 2026-09-16](https://vercel.com/changelog/typesafe-ai-jev-now-available-on-ai-gateway)).

### Limits
- Context: 64k tokens per request, 32k per question (state plus one question); an opt-in long-context header raises both to 262k, "slower and less accurate". Vercel's model page lists 32k. ([models](https://docs.typesafe.ai/models.md), [Vercel](https://vercel.com/ai-gateway/models/jev.md))
- Body size 32 MiB; HTTP 429 on rate limit, 529 on overload. ([errors](https://docs.typesafe.ai/api/errors.md))
- Rate limits are per organization (tokens/s and requests/s), visible in the console, no numbers published; higher limits via request form. ([models](https://docs.typesafe.ai/models.md))
- Choice: up to 255 options; Gateway score: 2-10 levels. ([Eden docs](https://www.edenai.co/docs/v3/llms/decisions.md), [Vercel OpenAI-compatible docs](https://vercel.com/docs/ai-gateway/sdks-and-apis/openai-decisions))

### Input modalities (contradictory across sources)
- TypeSafe direct: text, JSON, PNG/JPEG images; images are a "dreamer-ready preview, not production-ready", off by default, need the `TypeSafe-Danger-Zone-Images: true` header; PNG/JPEG only, 280-1568 px per side, 16 MiB per image, no resizing, EXIF stripped, whole request fails if any image fails. Jev 1.13 and 1.14 both accept images. ([images](https://docs.typesafe.ai/images.md), [models](https://docs.typesafe.ai/models.md))
- Vercel 1.14 changelog (2026-10-15): images in `state` as a file part or `input_image`. ([changelog](https://vercel.com/changelog/typesafe-ais-jev-1-14-isnow-available-on-ai-gateway))
- But Vercel's OpenAI-compatible Decisions doc (last updated 2026-10-08) says text-only models "such as `typesafe-ai/jev`" return 400 for images. ([doc](https://vercel.com/docs/ai-gateway/sdks-and-apis/openai-decisions)) That page predates the 1.14 changelog and may be stale, but I could not reconcile it. **Do not plan on image input via the Gateway until tested.**
- Implication: for v1, feed Jev structured text/JSON about Pieces (category, color name, material, formality, season, pattern), not photos. Photo understanding stays with the import pipeline (a multimodal LLM), a separate ticket.

### Versioning and pinning
- `jev-latest` -> `jev-1.14.0`; `jev-preview` also -> `jev-1.14.0` (no preview build). Aliases move, so answers can change without code changes; versioned IDs (`jev-1.13.0`, `jev-1.14.0`) are accepted and recommended for stable confidence thresholds. Each response reports the versioned ID that answered. No deprecation policy is documented. ([models](https://docs.typesafe.ai/models.md))
- Vercel Gateway: model page ID is `typesafe-ai/jev`; the 1.14 changelog references `typesafe-ai/jev-1.14.0`. Whether the Gateway keeps older versioned IDs is not documented (unverified).
- Jev is not fine-tuned on customer data; all accounts share the same weights. Customer data is not trained on; ZDR is enterprise-only on TypeSafe's side. ([models](https://docs.typesafe.ai/models.md), [legal](https://docs.typesafe.ai/legal.md)) Legal page does not cover data residency or subprocessors.

### Known weaknesses relevant to fashion
From TypeSafe's own ["model jaggedness"](https://docs.typesafe.ai/model-jaggedness.md) page:
- Reads instructions literally; negations and implied conditions are error-prone. Write exact, narrow questions.
- Bad with numbers and counting; Score outputs are "not numerically calibrated" and should not be interpolated into exact values. Pass named buckets (e.g. "navy", "warm"), never RGB/hex.
- Prefers the first Choice option; shuffle and check consistency.
- Large irrelevant state distracts; adversarial text in state can steer answers (relevant if imported product names/descriptions go into state).
- Calibration holds across groups of predictions, "does not guarantee that an individual answer is correct" ([System One](https://docs.typesafe.ai/concepts/system-one.md)). Vercel likewise says to calibrate against labeled examples ([changelog](https://vercel.com/changelog/typesafe-ai-jev-now-available-on-ai-gateway)).
- No documentation or cookbook covers fashion or style judgment; whether Jev has good taste is unknown.

## 2. Gateways serving Jev

| Route | Endpoint / ID | Notes |
|---|---|---|
| TypeSafe direct | `POST https://api.typesafe.ai/v1/systemone`, `jev-1.14.0` | Pinning, images (preview), long context; own key, own rate limits |
| Vercel AI Gateway | `POST https://ai-gateway.vercel.sh/v1/decisions` (OpenAI-compatible), model `typesafe-ai/jev`; also `/typesafe` base URL for the TypeSafe SDK, AI SDK `experimental_decide` | Logs, budgets, ZDR/no-training per request, BYOK, decision fallbacks; providers listed: typesafe-ai, digitalocean. `/v1/evaluate` is deprecated. AI SDK `evaluate` was renamed `decide` (experimental). ([Decision docs, updated 2026-10-07](https://vercel.com/docs/ai-gateway/modalities/decision), [changelog 2026-09-21](https://vercel.com/changelog/ai-gateway-now-supports-typesafe-clients-and-an-http-api-for-jev)) |
| Eden AI | `POST /v3/alpha/decisions`, model `typesafe/jev-latest` or `typesafe/jev-preview`, up to 7 fallbacks | Docs state **Alpha**: shapes, IDs and pricing "can change without a deprecation period", "risky to put on a critical path"; shows `jev-1.13.0` as concrete answering version. ([Eden docs](https://www.edenai.co/docs/v3/llms/decisions.md)) |

Vercel's **decision fallbacks** are useful: rerun a request with another model when Choice/Score confidence is low or a Noul probability sits in an uncertain band; both stages billed and latency adds. A language-model fallback returns no confidence or probabilities. ([Decision Fallbacks](https://vercel.com/docs/ai-gateway/models-and-providers/decision-fallbacks))

Note: the Gateway pattern is not unique to Jev. Vercel lists other decision models behind the same `/v1/decisions` API: OpenAI `openai/gpt-6-luna-decisions` ([2026-10-07](https://vercel.com/changelog/openai-decisions-api-now-available-on-ai-gateway)), Microsoft `microsoft/microsoft-decision-1`, text input ([2026-10-09](https://vercel.com/changelog/microsoft-decision-1-now-available-on-ai-gateway)), plus Perplexity Decider, Liquid AI d1 and others named in Vercel's related-pages lists. I did not evaluate their quality or price. The OpenAI-compatible endpoint means swapping the decision model is mostly a model-ID change.

## 3. Alternatives

### (a) Scoring and ranking Outfits
- **Gemini via Firebase AI Logic / Agent Platform (formerly Vertex AI):** Firebase AI Logic supports structured JSON output, image input, App Check and default per-user rate limits ([Firebase AI Logic](https://firebase.google.com/docs/ai-logic)). Current stable models include `gemini-3.5-flash-lite` and `gemini-3.8-flash`; up to 1,000 images per request, 7 MB per image ([models](https://firebase.google.com/docs/ai-logic/models)). Gemini API prices per 1M tokens: `gemini-3.5-flash-lite` $0.30 in / $2.50 out; `gemini-3.8-flash` $0.75/$3.75 through 2026-12-31, then $1.50/$7.50 ([pricing](https://aistudio.google.com/docs/pricing)). That is roughly 7x more input cost than Jev plus paid output tokens, and slower (generation), but it sees photos and explains itself. No calibrated probabilities (self-reported scores are not calibrated).
- Caveat vs ADR-0001: Firebase AI Logic is a client-side SDK path (App Check + per-user limits are its selling point). Domain logic lives in the backend, so the backend should call the Gemini API or Agent Platform server-side instead (or via Genkit); check whether per-user limits can be reused there (not verified).
- **Other decision models via the Gateway** (OpenAI, Microsoft, ...): same API shape, not evaluated here.
- **Pure code scoring** (color/formality/season rules): free, deterministic, explainable; the right baseline and the place for anything numeric. Jev should only add the taste judgments rules cannot express.

### (b) Embeddings for semantic search over the Wardrobe
- **`gemini-embedding-2`** (stable, updated April 2026): text, image, video, audio, PDF in one shared embedding space; 8,192-token input; up to 6 images per request; output 128-3072 dims (default 3072; 768/1536/3072 recommended); task type is a text prefix, not a parameter. Price: text $0.20/Mtok, image $0.45/Mtok (about $0.00012 per image); batch half price. Free tier exists but free-tier data "may be used to improve Google's products", so use the paid tier for users' photos. ([embeddings docs](https://aistudio.google.com/docs/embeddings), [pricing](https://aistudio.google.com/docs/pricing))
- `gemini-embedding-001`: text only, 2,048-token input, $0.15/Mtok. `multimodalembedding@001` (Agent Platform): 1408 dims, text/image/video shared space; `gemini-embedding-2` on Agent Platform uses global quotas, and a preview variant's regional quotas "might be deleted" ([Agent Platform multimodal embeddings](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-multimodal-embeddings)).
- **Firebase AI Logic does not support embeddings** ([models page](https://firebase.google.com/docs/ai-logic/models)), so embeddings must be called from the backend (Cloud Functions via Gemini API or Agent Platform).
- **Storage and query:** Firestore vector search: `flat` index, max 2048 dims (so use 768 or 1536, not 3072), `findNearest` with COSINE/EUCLIDEAN/DOT_PRODUCT, pre-filter with `where` needs a composite index, max 1000 results (Standard edition), no real-time snapshot listeners; SDKs are Python, Node.js, Go, Java only (no mobile SDKs, which matches ADR-0001 backend-only search) ([Firestore vector search](https://firebase.google.com/docs/firestore/vector-search)). The page does not state GA/preview or pricing; check before committing.
- Embedding Pieces: text description and photo can go into one embedding space, so a text query ("something warm for a wedding") can match photos of Pieces. Whether search quality is good enough on clothing photos is untested.
- **Jev as re-ranker:** TypeSafe's cookbook re-ranks BM25 shortlists with one Noul per (query, candidate) pair and improved top-5 hit rate from 15% to 57% on a legal corpus; it needs a shortlist containing the answer ([re-ranking cookbook](https://docs.typesafe.ai/cookbooks/rerank_typesafe.md)). Optional second stage for Vestis; not needed for v1.

## 4. Recommendation

1. **Outfit scoring: Jev, via a `DecisionModel` port in the backend**, with a code-only baseline score and an eval set.
   - Pipeline: candidates from graph-based generation, then Jev Noul/Score questions per dimension (e.g. "do these colors work together?", "is the formality consistent?", "fits the occasion/weather?"), then weighted sum in code, using TypeSafe's composite-scoring pattern ([patterns](https://docs.typesafe.ai/patterns/composite-scoring.md)).
   - State: compact JSON of Piece attributes with named buckets, no photos, no raw numbers; one judgment per question; shuffle Choice order.
   - Why: cheapest option by an order of magnitude, calibrated probabilities enable thresholds and escalation, per-request cost is predictable for the free app (map's cost guardrails item).
2. **Access route: start on Vercel AI Gateway `/v1/decisions`** (OpenAI-compatible, budgets and logs, ZDR/no-training, easy swap to other decision models or a Gemini fallback), **but verify first** that it can pin a versioned Jev ID; if not, call TypeSafe direct with `jev-1.14.0`. Never rely on `jev-latest` for thresholds. Avoid Eden AI for production (alpha). The Gateway adds a second vendor to the data path; weigh against GDPR (map item) and the unpublished data-residency terms of TypeSafe.
3. **Semantic search: `gemini-embedding-2` at 768 or 1536 dims, stored in Firestore vector search**, called from Cloud Functions on the paid tier. Embed a text rendering of each Piece plus its photo. Re-ranking with Jev is a later option.
4. **Fallback for scoring if Jev disappoints or access is withdrawn:** a Gemini Flash-Lite structured-output call from the backend, behind the same port. The Gateway's decision fallbacks can also escalate low-confidence cases.

## 5. Open risks and unknowns

- **Early access / maturity:** Jev is two months old, "early access"; Gateway `decide` API is explicitly experimental and has already been renamed (evaluate -> decide); `/v1/evaluate` deprecated. Expect churn; keep the adapter thin.
- **Taste quality unknown:** no evidence Jev judges outfits well. Need a labeled set of outfits (good/bad, by occasion) and a measured accuracy before launch. Calibration is group-level, so thresholds need our data.
- **Image support contradiction:** Vercel docs disagree with each other and with TypeSafe's preview-only status; test before relying on any image input.
- **Versions:** Gateway pinning, deprecation policy and `jev-1.13.0` retention are undocumented.
- **Rate limits and SLA:** per-org limits are unpublished; no SLA found. Free public app at scale could hit them; needs a quota design.
- **Latency:** only vendor claims (0.114 s example; 70-500 ms in secondary write-ups). Measure.
- **Pricing durability:** $0.042/Mtok may be introductory; Gemini Flash prices are changing on 2027-01-01 (3.8-flash doubles).
- **Prompt injection:** imported product text in `state` can steer Jev (jaggedness #6); sanitize or exclude free text.
- **Firestore vector search:** GA status and pricing not confirmed in the page read; no mobile SDK (fine under ADR-0001); 2048-dim cap forces truncation of Gemini's 3072 default.
- **Free-tier data use:** Google may train on free-tier embedding input; use paid tier.

## Sources (primary)

- TypeSafe: https://typesafe.ai ; https://docs.typesafe.ai/llms.txt ; /models.md ; /changelog.md ; /images.md ; /model-jaggedness.md ; /api/errors.md ; /legal.md ; /concepts/system-one.md ; /patterns/composite-scoring.md ; /cookbooks/rerank_typesafe.md ; /introduction/quickstart.md
- Vercel: https://vercel.com/ai-gateway/models/jev.md ; https://vercel.com/docs/ai-gateway/modalities/decision ; https://vercel.com/docs/ai-gateway/sdks-and-apis/openai-decisions ; https://vercel.com/docs/ai-gateway/models-and-providers/decision-fallbacks ; changelogs of 2026-09-16, 2026-09-21, 2026-10-07, 2026-10-09, 2026-10-15 (linked inline)
- Eden AI: https://www.edenai.co/docs/v3/llms/decisions.md
- Google: https://firebase.google.com/docs/ai-logic ; https://firebase.google.com/docs/ai-logic/models ; https://firebase.google.com/docs/firestore/vector-search ; https://aistudio.google.com/docs/embeddings ; https://aistudio.google.com/docs/pricing ; https://docs.cloud.google.com/vertex-ai/generative-ai/docs/embeddings/get-multimodal-embeddings
- Not primary, used only as pointers (not relied on for claims): web-search summaries of launch coverage (e.g. the 70-500 ms latency figure).
