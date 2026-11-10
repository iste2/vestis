# Firebase data store and vector search

Ticket: iste2/vestis#5. Research date: 2026-11-10. Sources are first-party (Firebase / Google Cloud docs and blog, FlutterFire repo, pub.dev). Where a claim is my inference rather than stated by a source, it is marked **(inference)**.

## Question

Which Firebase data store fits Vestis: Cloud Firestore or Firebase Data Connect (Cloud SQL for PostgreSQL)? Criteria: per-user isolation, modelling pieces/outfits/wears/graph edges, querying for outfit generation, offline in Flutter, vector search (Firestore vector search vs pgvector), image storage, cost at scale for a free public app.

## Answer in short

**Recommendation: Cloud Firestore (Standard edition), with Cloud Storage for Firebase for images and vector search executed server-side.** Data Connect is a credible alternative, not a mistake; it loses mainly on cost shape (always-on Cloud SQL instance), maturity of the Flutter client, and fit with ADR-0001's backend-centred design, where the relational power it adds is not needed on the client path.

Naming: Firebase Data Connect was renamed **Firebase SQL Connect** at Cloud Next 2026 (blog post dated 2026-04-22; the APIs, CLI commands and the `firebase_data_connect` Flutter package keep the old name). It reached GA on 2025-04-09 under the old name.

## Findings per criterion

### 1. Per-user data isolation

- **Firestore:** Security Rules are evaluated per client request before any data is read or written; if any requested path is denied the whole request fails. Rules combine with Firebase Authentication for user-based access. Server client libraries (Admin SDK, Cloud Functions) **bypass all rules** and use IAM instead. Rule changes take up to a minute to apply to new queries, up to 10 minutes for active listeners. ([Security Rules get started](https://firebase.google.com/docs/firestore/security/get-started)). The per-user pattern (`request.auth.uid` matched against an owner field or a `users/{uid}/...` path) is standard but the fetched page does not show it; see the structuring/conditions guides linked from it.
- **SQL Connect:** Clients can only call operations that carry an `@auth` directive; operations without one default to `NO_ACCESS`. Levels: `PUBLIC`, `USER_ANON`, `USER`, `USER_EMAIL_VERIFIED`, `NO_ACCESS`. Isolation is done by filtering on `auth.uid` in the operation (`authorUid: {eq_expr: "auth.uid"}`, owner set on create via `authorUid_expr: "auth.uid"`). Docs warn: `USER` level with no ownership filter behaves like `PUBLIC`. App Check attestation is supported. ([Authorization and security](https://firebase.google.com/docs/sql-connect/authorization-and-security))
- Verdict: both give solid per-user isolation. Firestore enforces it declaratively and uniformly on every query; SQL Connect enforces it per operation, so each operation must remember the filter. Slight edge Firestore.

### 2. Modelling pieces, outfits, wears, graph edges

- Firestore limits relevant to modelling: document max 1 MiB; 40,000 index entries per document; subcollection depth 100; 200 (no billing) / 1,000 (billing) composite indexes per database. ([Quotas](https://firebase.google.com/docs/firestore/quotas)) Standard edition requires an index for every query (single-field indexes are automatic). ([Compare editions](https://firebase.google.com/docs/firestore/compare-editions-for-native-mode))
- Firestore has no joins in Standard (Core) operations; Pipeline operations (including JOIN via subqueries) exist only in the **Enterprise** edition, described as GA in the Cloud Next 2026 post ([blog](https://firebase.blog/posts/2026/04/cloud-next-2026-announcements/)). Standard supports Core operations only.
- SQL Connect is relational (schema in GraphQL -> Postgres tables, foreign keys, joins in generated queries). Native SQL (launched 2026-03-24, status label not stated in the post) adds free-form SQL, multi-step CTEs, window functions, extensions; the post does not confirm recursive CTEs. ([Native SQL post](https://firebase.blog/posts/2026/03/fdc-native-sql/))
- **(inference)** Vestis's graph is small per user (a wardrobe of tens to low hundreds of pieces; edges are piece-to-piece compatibility). Per ADR-0001 generation runs once in the backend, so the backend can load a user's wardrobe and edges and traverse in memory; no in-database graph query is needed. Edges can be stored as an `edges` subcollection or as per-piece arrays/maps (mind the 1 MiB doc limit and 20 nesting depth). Relational joins would be nicer for wear statistics (cost per wear, unworn pieces) but those are also computable in backend code or by Firestore aggregation queries.
- Verdict: Postgres models this more naturally; Firestore is adequate given the data size and backend-side logic. Edge SQL Connect on modelling.

### 3. Querying for outfit generation

Backend-side (ADR-0001), so client query features matter little. Firestore Admin SDK reads of one user's wardrobe cost one read per document (see cost). SQL Connect bills per operation, not per row, so "load whole wardrobe + edges" is one operation. ([Pricing](https://firebase.google.com/docs/sql-connect/pricing)) Edge SQL Connect on per-request cost for read-heavy generation; Firestore edge on zero fixed cost.

### 4. Offline support in Flutter

- **Firestore:** Persistence is on by default on Android/iOS; writes made offline are queued and synced on reconnect (last write wins per document); default cache threshold 100 MB; offline queries run only over cached documents; Flutter configures it via `Settings(persistenceEnabled:, cacheSizeBytes:)`. ([Offline data](https://firebase.google.com/docs/firestore/manage-data/enable-offline)) Mature, years in production. `cloud_firestore` is at 7.1.0 (2026-10-23, [pub.dev](https://pub.dev/packages/cloud_firestore)).
- **SQL Connect:** Optional client cache configured in `connector.yaml`; default storage on Android/iOS is `persistent`, web is memory only; default `maxAge` is 0 (always fetch unless `CACHE_ONLY`); `subscribe()` returns cached data first. ([Flutter SDK](https://firebase.google.com/docs/sql-connect/flutter-sdk)) Realtime updates need `firebase_data_connect` >= 0.3.0 and CLI >= 15.14.0 ([Realtime](https://firebase.google.com/docs/sql-connect/realtime)). This is a read cache; the docs fetched do not describe an offline **write** queue. The package is at 0.4.1 (2026-10-23, [pub.dev](https://pub.dev/api/packages/firebase_data_connect)), had a breaking change in 0.4.0, and its README still labels the product "Public Preview" (stale vs. the GA announcement, but a signal of churn).
- Verdict: Firestore clearly ahead. Offline behaviour is a map-level open item ("Offline behaviour and sync in the app"); Firestore gives a working baseline for free.

### 5. Vector search

- **Firestore (Standard):** `findNearest` over a vector field; distance measures EUCLIDEAN, COSINE, DOT_PRODUCT; **max 2048 dimensions**; **index type `flat` only**; at most 1000 results; pre-filtering with field filters needs a composite vector index; no real-time listener support; Firestore does not generate embeddings (you call an external service and store the vector). Documented client SDKs: **Python, Node.js, Go, Java only**. ([Vector search](https://firebase.google.com/docs/firestore/vector-search)) Billing: one read per returned document plus one read per batch of 100 kNN index entries scanned ([pricing](https://cloud.google.com/firestore/pricing), via search summary).
- **Flutter client:** `VectorValue` (store/read vectors) was added in `cloud_firestore` 5.6.0; `findNearest` exists in the Flutter plugin only as a **Pipeline stage** (`pipeline.dart`, e2e test `pipeline_find_nearest_e2e.dart`, 7.1.0 added returning the distance) ([changelog](https://github.com/firebase/flutterfire/blob/main/packages/cloud_firestore/cloud_firestore/CHANGELOG.md)). Pipelines are Enterprise-edition operations ([compare editions](https://firebase.google.com/docs/firestore/compare-editions-for-native-mode)). So with Standard edition, vector queries cannot be run from the Flutter client; they must run server-side. This matches ADR-0001 anyway.
- **SQL Connect:** `Vector` field type maps to pgvector `vector` (single-precision), declared with `@col(size: N)`; `_embed` server value calls Vertex AI embeddings (examples use `text-embedding-005`, 768 dims) in mutations and queries, or supply your own; a `<plural>_<field>_similarity` query is generated with `method` (COSINE, INNER_PRODUCT, L2), `within`, `where`, `limit`. Vertex AI integration only on certain Cloud SQL locations; embedding model versions are incompatible with each other; Vertex AI usage is billed. ([Vector similarity search](https://firebase.google.com/docs/sql-connect/solutions-vector-similarity-search)) The page does not mention HNSW/IVFFlat index creation; whether per-user `@auth` filters combine with the similarity query is not addressed (its examples use `PUBLIC`). **(inference)** `where` should allow an `auth.uid` filter, but this needs a spike.
- Per-user wardrobe size makes both fine: flat/brute-force over one user's few hundred embeddings is trivial. **(inference)** With a Firestore composite vector index prefiltered on owner id, scanned index entries should be limited to that user's vectors, keeping read cost low; verify in a spike.
- Verdict: both capable; SQL Connect has the nicer built-in embedding pipeline, Firestore requires an embedding call in backend code (needed anyway for image/text embeddings and the decision-model/AI work).

### 6. Image storage

Cloud Storage for Firebase is the answer for either store (neither holds blobs). It requires the **Blaze** plan; new `*.firebasestorage.app` buckets include 5 GB-months stored, 100 GB/month download, 5K uploads and 50K downloads per month free, only in us-central1, us-west1, us-east1 ([Firebase pricing](https://firebase.google.com/pricing)). Security Rules expose `request.auth` and can validate size and `contentType` on write ([Storage security](https://firebase.google.com/docs/storage/security)); a per-user path rule is the standard pattern but isn't shown on the fetched page. **(inference)** An EU bucket (likely desirable for body photos/GDPR) falls outside the free-quota regions, so storage and egress are paid from the first byte; budget for it. Because Blaze is required for Storage regardless, the "Spark-only, no card" benefit of the SQL Connect trial is moot for Vestis.

### 7. Cost at scale for a free public app

- **Firestore Standard** (us-central1): reads $0.30/M, writes $0.90/M, deletes $0.10/M, storage about $0.15/GiB-month; free daily quota 50K reads, 20K writes, 20K deletes, 1 GiB. Real-time listener updates bill as reads. Scales to zero; cost is purely usage-proportional. ([Firestore pricing](https://cloud.google.com/firestore/pricing), [Firebase pricing](https://firebase.google.com/pricing))
- **SQL Connect:** Cloud SQL instance from about $9.37/month (db-f1-micro class, region/config dependent) as a **fixed floor**, after a 3-month trial (Blaze: default config, 1 per project, 5 per billing account; Spark: 90 days, ~8.3K ops/day, then archived and deleted 90 days later). Operations: 250K/month free then $0.90/M, per operation not per row; egress 10 GiB/month free. Adding compute, private IP or a read replica ends the trial and bills. ([SQL Connect pricing](https://firebase.google.com/docs/sql-connect/pricing)) A single small instance is a scaling and availability ceiling; a public multi-user app will need a larger tier early (cost **inference**: scales in steps, not smoothly).
- Firestore Enterprise is priced differently (reads $0.05/M units in 4 KiB, writes $0.26/M units per KiB, storage $0.00032/GiB-hour) and needs no indexes ([Enterprise pricing](https://firebase.google.com/docs/firestore/enterprise/pricing)); not needed unless Pipelines/client vector search become a requirement.
- Auth: Identity Platform 50K MAU free then Google Cloud pricing; phone auth billed per SMS ([Firebase pricing](https://firebase.google.com/pricing)).
- Verdict: for an unproven free app, Firestore's pay-per-use is safer; SQL Connect's floor is small but real and its ceiling needs a resize.

## Comparison

| Criterion | Firestore Standard | SQL Connect (Postgres) |
|---|---|---|
| Per-user isolation | Declarative Security Rules; Admin SDK bypasses | Per-operation `@auth` + `auth.uid` filters; no-`@auth` = no client access |
| Modelling graph/relations | Documents, no joins (Standard) | Relational, joins, native SQL |
| Offline in Flutter | Mature read+write persistence | Read cache; no offline write queue documented; package 0.4.x |
| Vector search | Server SDKs only (Py/Node/Go/Java), flat, <=2048 dims, composite index for prefilter | pgvector via `Vector`, `_embed` via Vertex AI, COSINE/IP/L2 |
| Image storage | Cloud Storage (Blaze) | Cloud Storage (Blaze) |
| Cost shape | Usage-based, zero floor | Fixed instance floor (~$9.37+/mo) + $0.90/M ops |
| Maturity | Longest track record | GA since 2025-04, renamed 2026-04, fast-moving features |

## Open risks and spikes

1. **Vector search from backend code:** confirm a Cloud Functions / Cloud Run (Node or Python) `findNearest` with composite vector index prefiltered by owner (e.g. `ownerId` + wished/owned filter) works and measure index-entries-read per query for a 500-piece wardrobe.
2. **Embedding model/dimension choice** is owned by the AI-provider tickets; Firestore caps at 2048 dims. Switching the embedding model later means re-embedding (applies to both stores).
3. **Graph edges:** decide stored edges (subcollection) vs computed-on-demand; watch the 1 MiB document and 500-transform limits for hub pieces.
4. **Region/GDPR:** Firestore location is immutable after creation; Storage free quota only in three US regions. Needs the privacy ticket before creating the production project.
5. **Rules vs Admin SDK:** because generation, search and the MCP server run server-side and bypass Rules, ADR-0001's backend must enforce `uid` scoping itself in code; Rules protect only direct Flutter access.
6. **SQL Connect is not ruled out:** revisit if wear statistics/graph analytics become heavy and relational, or if the Flutter client's offline/realtime support matures (its status labelling is inconsistent across pages: README says "Public Preview", blog says GA since 2025-04).
7. **Not verified from primary sources:** exact Firestore vector query billing (taken from a search summary of the pricing page, which could not be fetched in full); SQL Connect HNSW index support; Firestore per-database write throughput limits.
