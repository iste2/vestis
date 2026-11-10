# Avatar generation and virtual try-on services

Research for issue #8. Researched 2026-11-10 against vendor documentation. Facts marked **(unverified)** could not be confirmed on a primary page. Vendor docs and prices change often; re-check before committing.

## Short answer

- A user can get an **avatar** by uploading one or a few full-body photos; no measurements are needed by any surveyed service. The services take a photo of a person and a photo of a product and return an image.
- **Multi-piece outfit try-on is the weak spot.** Neither Google's dedicated Virtual Try-On model nor FASHN documents combining several pieces in one request. Documented workarounds: chain single-piece calls (output becomes the next person image) or use a general Gemini image model with several reference images.
- Recommended v1 path: Google Virtual Try-On (`virtual-try-on-001`) on Vertex AI, run from the Firebase/Google Cloud backend, with chained calls for outfits, and Gemini image models as an experiment for one-shot outfits. FASHN is the fallback.

## 1. Google Vertex AI Virtual Try-On (`virtual-try-on-001`)

- **Status:** generally available since 2026-01-23; replaces `virtual-try-on-preview-08-04`, which was deprecated 2026-02-17 (update before 2026-03-19). Source: [Vertex AI release notes](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/release-notes). Dec 2025 update reduced latency and improved shoes, body-shape preservation and product fidelity (same page).
- **Inputs:** one person image plus product image(s); PNG or JPEG; model page states "Maximum 2 images per prompt" and 7 MB per file inline (the older page said 10 MB). Output aspect ratio and resolution match the input; 1 to 4 outputs per request. Source: [model page](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/vto/virtual-try-on-001), [how-to](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/image/generate-virtual-try-on-images).
- **Multi-piece:** the REST schema has a `productImages` array, but every documented example sends one product, and "max 2 images per prompt" suggests person + one product. Multi-garment in one call is **not documented** (checked via [VirtualTryOnModelInstance](https://docs.cloud.google.com/gemini-enterprise-agent-platform/reference/rest/Shared.Types/VirtualTryOnModelInstance) search result and how-to). Needs a direct API test.
- **Categories:** not listed; shoes are mentioned in release notes.
- **Cost:** $0.06 per output image (1 person image + 1 clothing image), per the Imagen section of the Vertex pricing page as seen in a search snippet about 350 days old; **not re-confirmed** against the current page, which could not be fetched in full ([pricing](https://cloud.google.com/vertex-ai/generative-ai/pricing)).
- **Latency:** not documented by Google (release notes only say "significantly reduced").
- **Regions:** includes EU regions (europe-west1, -west4, -west8, -west9, -southwest1, -north1) plus US, CA, APAC, ME. Pay-as-you-go and Provisioned Throughput; no batch.
- **Provenance:** C2PA Content Credentials listed as supported; SynthID not mentioned on that page.
- **Privacy/terms:** Google Cloud commits not to train on customer data without permission; request-response logging is off by default; prompts may be logged for abuse monitoring under the Cloud terms; implicit caching (Gemini models) up to 24 h. The data-governance page does not mention virtual try-on explicitly. Source: [Vertex data governance](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/data-governance).

## 2. Gemini image models ("Nano Banana" family)

Source: [Gemini image generation docs](https://aistudio.google.com/docs/image-generation) and [Gemini API pricing](https://aistudio.google.com/docs/pricing), fetched 2026-11-10. Also available on Vertex AI.

| Model ID | Notes | Price per image (standard, paid) |
|---|---|---|
| `gemini-nano-banana-2.1` | Recommended for new projects | $0.0336 (1K), $0.0504 (2K), $0.113 (4K) |
| `gemini-3.1-flash-image` | Nano Banana 2 | $0.067 (1K), $0.101 (2K) |
| `gemini-3.1-flash-lite-image` | Cheapest, weak on multi-reference | $0.0336 (1K), batch $0.0168 |
| `gemini-3-pro-image` | Premium, with thinking | $0.134 (1K/2K), $0.24 (4K) |
| `gemini-2.5-flash-image` | Legacy; shutdown 2027-03-15 | $0.039 |

- **Multi-piece:** these are general image editors with up to 14 mixed reference images (Nano Banana 2/2.1: up to 10 object and 4 character images; Pro: up to 6 objects, 5 characters). That makes one-shot outfit renders plausible (person + top + trousers + shoes). The docs contain **no try-on example and no fidelity guarantee**; garment and identity fidelity must be evaluated by us.
- **Latency:** no figures published ("Flash-level speed").
- **Watermark:** all outputs carry SynthID.
- **Privacy:** paid tier content is not used to improve Google products; the free tier is (for the listed models), and the free tier has no image-model quota. So production must use the paid tier or Vertex AI.
- Imagen is listed as shut down on the Gemini docs; Vertex release notes show Imagen GA endpoints deprecated 2026-03-24.

## 3. FASHN API

Sources: [try-on max](https://docs.fashn.ai/api-reference/tryon-max), [try-on v1.6](https://docs.fashn.ai/api-reference/tryon-v1-6), [pricing](https://fashn.ai/pricing), [data retention](https://docs.fashn.ai/api-overview/data-retention-privacy), [subprocessors](https://fashn.ai/legal/subprocessors).

- **Try-On Max:** `model_image` + one `product_image` (clothing, shoes, hats, jewelry, bags); optional text prompt; 1k/2k/4k; modes fast/balanced/quality; 1 to 4 images; latency about 10 s (fast, 1k), 25 s (balanced, 2k), 55 s (quality, 4k); 1 to 5 credits per image (balanced 1k = 2).
- **Try-On v1.6:** tops/bottoms/one-pieces; 864x1296 processing; about 5 to 17 s; 1 credit per image.
- **Price:** API credits $0.075 each, pay-as-you-go, $7.50 minimum, expire after 12 months, no free API credits. So v1.6 is about $0.075 per image and Max balanced-1k about $0.15 (derived, not listed as such).
- **Multi-piece:** not documented; single `product_image`.
- **Avatar:** FASHN lists a "model create" endpoint (generating a person image); inputs not verified.
- **Privacy:** no training on customer content unless opted in; URL inputs fetched only for processing; base64 inputs deleted at request end (1-day backstop); output CDN files deleted after 3 days; base64 outputs available 60 min. Subprocessors include Google, OpenAI, fal, BytePlus (Singapore/Malaysia/Indonesia), AWS, Cloudflare, Supabase (US); a DPA exists. Third-country processing makes GDPR work heavier than Vertex EU regions.

## 4. Open models

- CatVTON ([repo](https://github.com/Zheng-Chong/CatVTON)): about 0.9B parameters, around 8 GB VRAM at 1024x768, license CC BY-NC-SA 4.0.
- IDM-VTON ([repo](https://github.com/yisol/IDM-VTON)): CC BY-NC-SA 4.0, needs DensePose/parsing/OpenPose checkpoints; upper_body/lower_body/dresses.
- Both are **non-commercial licensed**, so unsuitable for a public app (even a free one, the app is a product), and would require GPU hosting. Not recommended. Licensing of any other open model must be checked individually (not surveyed).

## 5. Avatar definition options (glossary: Avatar = digital likeness used for try-on)

1. **Photo-as-avatar:** store one or more user-provided full-body photos; every service above takes such a photo directly. Simplest, most faithful, but the photo itself is the sensitive artifact.
2. **Generated canonical avatar:** produce a neutral standing full-body image (plain background, fitted clothes) from selfies/photos with a Gemini image model (or FASHN model create), then use it as the stable try-on base. Costs one render at setup, improves consistency across outfits and avoids repeated uploads of raw photos. Quality/identity fidelity not benchmarked here.
- Measurements: not an input to any surveyed service; body shape is inferred from the photo. Measurements would only matter for a separate size/fit feature.

## 6. Cost sketch (owner carries AI cost)

- Single-piece render: about $0.034 (Gemini 2.1 1K) to $0.06 (Vertex VTO) to $0.075 to $0.15 (FASHN).
- Outfit of N pieces by chaining: roughly N x the single-piece price (e.g. 4 pieces about $0.24 on Vertex VTO), plus compounding artifacts and latency N x per-call.
- One-shot outfit with Gemini image model: one call at about $0.034 to $0.134 depending on model, but fidelity unproven.

## Open risks

- Multi-piece support is undocumented by Google VTO and FASHN; chaining may degrade identity/garment fidelity. A fidelity spike with real pieces is needed.
- Current Vertex VTO price and quota are unconfirmed.
- Body photos are sensitive personal data (GDPR; possibly biometric-adjacent); retention, deletion, consent and minors policy are open (map item "Privacy and data protection"). Prefer EU regions on Vertex, avoid free-tier Gemini API, set logging off, delete inputs after render.
- Abuse (rendering other people's photos, nudity, minors) needs guardrails; per-user quotas needed for cost (map item "Cost and abuse guardrails"). Provisioned Throughput/quotas for VTO are not documented.
- Generated images carry SynthID/C2PA; store listings may require AI-content disclosure (not researched).
- Model churn: Google deprecates image models every few months (2.5 Flash Image shuts down 2027-03-15), so the backend should hide the provider behind one interface (consistent with ADR-0001).
