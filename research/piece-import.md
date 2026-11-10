# Importing pieces from product links and photos

Research for iste2/vestis issue #9. Research date: 2026-11-10. Not legal advice; legal points need confirmation by counsel before launch.

## Question

How do we turn a product link or a photo into a piece? Product links: schema.org `Product` JSON-LD, OpenGraph, shop APIs, bot protection, scraping law, image rights. Photos: cut-out, attribute extraction with a vision model, on-device vs cloud in Flutter/Firebase, cost per import.

## Short answer

1. **Product links:** fetch the page server-side (Cloud Function, user-triggered, one URL at a time), parse schema.org `Product` JSON-LD first, OpenGraph second, and fall back to a cheap LLM over the visible text/image for what is missing. Do not build crawlers, do not circumvent bot protection, and degrade to "add by photo / manually" when blocked.
2. **Photos:** extract attributes with Gemini through Firebase AI Logic (structured JSON output, App Check). Do the cut-out on-device (iOS Vision, Android ML Kit) rather than in a cloud image model, which costs roughly 20x more per import.
3. **Cost:** about 0.2 cent per link import and 0.15 cent per photo attribute extraction; a cloud cut-out adds about 3.4 to 6.7 cents. See "Cost per import".
4. **Legal:** a user-initiated fetch of a single product page is far from the bulk scraping that the TDM and database rules target, but copying shop images and breaching shop terms are real residual risks. Prefer storing metadata plus a link, and use the user's own photo as the piece image where possible.

## 1. Product links

### 1.1 What structured data shops publish

- schema.org `Product` has `name`, `image`, `brand`, `color`, `material`, `size`, `pattern`, `audience`, `category`, `gtin*`, and `ProductGroup` with `hasVariant`. Source: https://schema.org/Product (page footer: V30.1, 2026-09-16, development version).
- Google's merchant-listing guidance recommends exactly the properties a wardrobe needs: `color` (text), `size`, `material`, `pattern`, `audience.suggestedGender`, `audience.suggestedMinAge/MaxAge`, `brand.name`, `gtin*`, `image` (high resolution), and `ProductGroup`/`isVariantOf` for variants. Source: https://developers.google.com/search/docs/appearance/structured-data/merchant-listing
- Google's basic product-snippet page requires only `name` plus one of `review`/`aggregateRating`/`offers`, and lists JSON-LD, RDFa and Microdata. It does not list color/size/material. Source: https://developers.google.com/search/docs/appearance/structured-data/product-snippet
- Consequence: JSON-LD is common for large fashion shops because of Google rich results, but `color`, `material`, `pattern` are only recommended, so coverage will be uneven. Material and category often exist only in free text. (Coverage across shops is not measured here; sample real shops in the implementation slice.)

### 1.2 Extraction pipeline (recommended)

1. Backend fetches the URL (see ADR-0001: logic in the backend, app is a thin client).
2. Parse all `<script type="application/ld+json">` blocks (including `@graph`), pick `Product` or `ProductGroup`; resolve `offers`, `brand`, `image`.
3. Merge OpenGraph (`og:title`, `og:image`, `og:description`) for gaps.
4. For missing attributes (category, material, pattern), pass title, description and the main image to a cheap Gemini model with a JSON schema constrained to Vestis' piece attributes.
5. Return a draft piece to the app; the user confirms or edits before saving (import is a suggestion, not a commit).

Shop APIs and affiliate feeds (per-shop integrations) are not worth it for v1: they are per-merchant, contract-bound and do not cover "any link". Revisit only if links are later a monetisation or quality issue.

### 1.3 Bot protection

- robots.txt is not an access-control mechanism: RFC 9309 states "These rules are not a form of access authorization" (IETF Standards Track, September 2022). Source: https://www.rfc-editor.org/rfc/rfc9309.html . It is still the signal that the TDM-reservation and good-citizen arguments hang on, so honour it for automated fetches.
- Large fashion shops commonly sit behind CDN bot management; a datacenter IP fetch will sometimes get a challenge page. Treat this as an expected failure mode (not measured here). Do not build evasion (headless-browser farms, residential proxies): that undermines the legal position in section 1.4 and is a maintenance sink. Fallbacks: (a) the share-sheet flow where the app (on the user's device, user's IP) fetches the HTML and sends it to the backend; (b) the user adds a screenshot, which is the photo flow (section 2).

### 1.4 Legal limits (EU/Germany)

Not legal advice; sources are primary where noted.

- **Copyright: TDM exception, section 44b UrhG.** Reproductions of lawfully accessible works for text and data mining (automated analysis to derive information) are permitted if the rightsholder has not reserved the use; for online works the reservation is effective only if machine-readable (section 44b(3)). Inserted by law of 31 May 2021, in force 7 June 2021. Source: https://www.gesetze-im-internet.de/urhg/__44b.html (server returned 503 during research; wording verified via https://dejure.org/gesetze/UrhG/44b.html). It implements Art. 4 of Directive (EU) 2019/790: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32019L0790 (Recital 18 says reservation should be machine-readable for online content).
- **Case law on the reservation:** OLG Hamburg, 10 December 2025, 5 U 104/24 (Kneschke v LAION) upheld the 2024 LG Hamburg ruling; secondary reports disagree on how far natural-language opt-outs count, and the case was reported as headed for the Federal Court of Justice. I could not read the judgment itself; do not rely on this point without checking it. Reports: https://heise.de/-11111575 , https://www.technologyslegaledge.com/2025/12/robert-kneschke-v-laion-judgment-of-10-december-2025-ref-5-u-104-24/
- Note that TDM is a poor fit for our use: we do not analyse a corpus, we extract one product's facts for the user who pasted the link. Facts and short metadata (colour, size, brand) are not copyrightable in themselves; the risk sits in copying product photos and descriptive texts.
- **Contract / terms of use.** CJEU, Ryanair v PR Aviation (C-30/14, 15 Jan 2015): the Database Directive does not apply to a database protected neither by copyright nor by the sui generis right, so contractual limits on use (e.g. "no screen-scraping") are not precluded "without prejudice to the applicable national law". Source: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:62014CJ0030 . Consequence: shop terms forbidding automated retrieval can bind, mainly where the scraper has accepted them (click-wrap); a single user-initiated fetch without account carries low practical risk but is not zero.
- **Sui generis database right** (Directive 96/9/EC) targets extraction/reutilisation of a substantial part of a database; one product page is not that. Do not bulk-crawl catalogues.
- **Image rights.** Shop product photos are protected as works or as photographs (section 72 UrhG neighbouring right) and are the shop's or photographer's property. Hosting a copy in Vestis' storage and showing it to the user is a reproduction not clearly covered by 44b or by a private-copy argument when done by a commercial backend. Mitigations, in order of preference: (1) default the piece image to the user's own photo; (2) hotlink/proxy-display the shop image only transiently and keep the source URL; (3) if copied, store per-user only, never share across users, delete on piece deletion, and offer a takedown path. Needs a lawyer's view before store launch.
- **GDPR** is only touched if a product page contains personal data (reviews); extraction should ignore reviews.

## 2. Photos

### 2.1 Attribute extraction with a vision model

- **Firebase AI Logic** is the managed path from Flutter: accepts images with text, supports structured JSON output, and App Check protects the API; the Gemini API key stays server-side. Source: https://firebase.google.com/docs/ai-logic
- Two backends: Gemini Developer API (Spark no-cost or Blaze, explicit rate limits, no location selection) and the Vertex-based "Agent Platform Gemini API" (Blaze only, location selectable, default `global`). Per-user default limit is 100 requests per minute. Source: https://firebase.google.com/docs/ai-logic/faq-and-troubleshooting
- **App Check becomes mandatory from 2 November 2026** (auto-enforced; delay possible only until 21 January 2027). Production attestation: App Attest/DeviceCheck on Apple, Play Integrity on Android. Same source. This is already in force today and must be part of the scaffolding.
- Recommended models listed by Firebase on 2026-11-10: `gemini-4-argon` (Blaze), `gemini-3.8-flash`, `gemini-3.5-flash-lite`. `gemini-2.5-*` is being retired (Agent Platform 20 Oct 2026; Developer API rerouted from 1 Dec 2026). The page recommends Remote Config or server prompt templates so model names can change without an app release. Source: https://firebase.google.com/docs/ai-logic/models . Take that advice: never hard-code model IDs in the app.
- Alternatively call Gemini from a Cloud Function (needed anyway for link import; ADR-0001). For photo extraction either is fine; the Cloud Function keeps prompts, schema and quotas in one place, matching ADR-0001 and the open "cost and abuse guardrails" question on the map.

### 2.2 Cut-out (background removal)

| Option | Where | Status / facts | Cost |
|---|---|---|---|
| Apple Vision `VNGenerateForegroundInstanceMaskRequest` | on-device iOS | iOS 17+ (from Apple's API knowledge; the docs page needs JS and could not be read, so verify) | free |
| ML Kit Subject Segmentation | on-device Android | Beta, no SLA, Android only, static images, about 200 ms on Pixel 7 Pro, foreground mask and bitmap. Source: https://developers.google.com/ml-kit/vision/subject-segmentation . Flutter plugin `google_mlkit_subject_segmentation` 0.3.1 is Android-only. Source: https://pub.dev/packages/google_mlkit_subject_segmentation | free |
| Gemini image editing ("Nano Banana") via Firebase AI Logic | cloud | Models `gemini-nano-banana-2.1`, `gemini-3.1-flash-image`, `gemini-3.1-flash-lite-image`, `gemini-3-pro-image`; all require billing; Imagen is shut down (models page above) | about $0.034 to $0.067 per image (below) |

Implications: there is no single cross-platform Flutter package for both OSs; on-device means a small platform channel (Vision on iOS, ML Kit on Android) behind one Dart interface. Generative editing is not a true matte (it may redraw the garment), which matters for a wardrobe where colour and cut fidelity are the point (not tested here). Recommended: on-device cut-out as default, optional cloud fallback only for failed cases. Cut-out quality on flat lays, hangers and worn garments should be verified in a prototype (open risk).

## 3. Cost per import

Prices (Standard tier, USD per 1M tokens) from https://aistudio.google.com/docs/pricing , read 2026-11-10:

- `gemini-3.5-flash-lite`: $0.30 in / $2.50 out. `gemini-3.1-flash-lite`: $0.25 / $1.50. `gemini-3.8-flash`: $0.75 / $3.75 until 31 Dec 2026, then $1.50 / $7.50.
- Images are billed at the model's input token rate; the page gives no per-image price for text models.
- `gemini-3.1-flash-lite-image`: $0.0336 per 1K image output (input $0.25/M). `gemini-3.1-flash-image`: $0.067 (1K), $0.045 (0.5K).
- Free tier exists for text models in the Developer API but is not usable for a public app at scale and its data may be used to improve Google products (page notes this for some models); plan on paid tier.
- The pricing page notes `gemini-2.5-flash-image` shutting down 15 March 2027 while the Firebase models page says 2 October 2026 on Firebase: avoid it either way.

Estimates (my token assumptions, not measured; an image is assumed to cost about 1,000 to 1,500 input tokens):

| Import | Assumptions | Cost with `gemini-3.5-flash-lite` |
|---|---|---|
| Link, JSON-LD complete | no LLM call | about $0 |
| Link, LLM gap-fill | 5,000 in, 300 out | about $0.0023 |
| Photo attributes | 1,500 + 500 prompt in, 400 out | about $0.0016 |
| Photo cut-out in cloud | one 1K image out | about $0.034 to $0.067 |
| Photo cut-out on-device | none | $0 |

Same photo extraction with `gemini-3.8-flash` after 1 Jan 2027 prices: about $0.0063. Even at 100 imports per user a month, text-model cost stays well under $1 per user; the cloud cut-out is what would dominate and is the main reason to keep it on-device. Each import should be metered against the per-user quota decided in the (open) guardrails ticket.

## 4. Recommendation

- **Links:** server-side fetch, JSON-LD then OpenGraph then LLM gap-fill, user confirms draft. No crawling, no evasion, honour robots.txt, fall back to photo/screenshot/manual. Default piece image to the user's own photo; treat shop images as the legally sensitive part.
- **Photos:** Firebase AI Logic (or Cloud Function) with Gemini Flash-Lite-class model and a JSON schema; on-device cut-out (platform channel) with cloud image editing only as fallback; model IDs through Remote Config.
- **Scaffolding dependencies:** App Check with production attestation (enforced from 2 Nov 2026), Blaze plan, per-user quotas.

## 5. Open risks and questions

- Shop-image reproduction rights: need legal review; decide copy vs. transient display (candidate decision ticket).
- Terms-of-use of individual shops and bot protection failure rate: sample 20 to 30 real shops (DE fashion) before committing to the pipeline; this is a good first implementation spike.
- On-device cut-out quality for garments and the Dart-side abstraction over two native APIs: prototype.
- OLG Hamburg 5 U 104/24 follow-up (BGH) could change TDM reservation rules; the judgment text was not read.
- Apple Vision documentation could not be read (JS-only page); iOS 17 requirement is unverified against Apple's page.
- Token costs per image are assumed, not measured; measure in the first slice.
- Data-residency: only the Agent Platform backend offers location selection; matters for the open privacy ticket (body photos are separate, but piece photos may show people).
