# Backend runtime and language for shared domain logic

Ticket: iste2/vestis#4. Researched 2026-11-10. Context: ADR-0001 (domain logic lives once in the backend; Flutter app and MCP server are both clients).

## Short answer

Host the domain logic as **one HTTP service in TypeScript (Node.js 22 or 24)**, deployed as a **Cloud Functions for Firebase 2nd gen HTTPS function** (which is a Cloud Run service underneath) wrapping a portable HTTP app, so it can move to a plain Cloud Run container without a rewrite. Do **not** pick Dart for the backend in v1: Dart on Cloud Functions for Firebase is experimental (May 2026), and only HTTPS/callable triggers deploy. Share the model contract with Flutter through a schema (e.g. OpenAPI / JSON Schema) with generated Dart types instead of a shared Dart package. Revisit Dart when `firebase_functions` leaves experimental status. See "Risks" for what could flip this.

## Findings (with sources)

### Cloud Functions for Firebase: languages
- Officially supported: Node.js 20 and 22 (TypeScript is an authoring option), Python 3.10-3.13 (3.13 default). Dart is not mentioned on the main page. Source: https://firebase.google.com/docs/functions/get-started (fetched 2026-11-10).
- 2nd gen functions are deployed as Cloud Run services (built with Cloud Build); up to 1,000 concurrent requests per instance, up to 60 min timeout for HTTP functions, up to 16 GiB / 4 vCPU. Source: https://firebase.google.com/docs/functions/version-comparison

### Dart on Cloud Functions for Firebase: experimental
- Announced 2026-05-06 (Firebase blog, Kevin Moore and Rody Davis) as experimental behind a CLI flag: `firebase experiments:enable dartfunctions`. Initial release supports HTTPS and callable functions. Dart is AOT-compiled to native binaries (blog claims reduced cold starts, gives no benchmark). Data models and validation rules can live in a shared Dart package used by Flutter and backend. Source: https://firebase.blog/posts/2026/05/dart-functions-exp
- Getting-started page: needs Dart SDK >= 3.9, Firebase CLI >= 15.15.0; deploy needs Blaze. Feature "may change without notice". Emulator supports only `onRequest` and `onCall` (other triggers run in emulator but cannot be deployed). The CLI compiles Dart locally and uploads the binary to Cloud Run functions. Limitations: `onCall` functions cannot be invoked by name via `httpsCallable` (must use `httpsCallableFromURL` with the Cloud Run URL); Dart functions are not listed in the Firebase console (only the Cloud Run page in the Google Cloud console). The Dart Admin SDK (`firebase_admin_sdk`) is also experimental. Source: https://firebase.google.com/docs/functions/start-dart
- Repo README: only HTTPS is described as production-ready; Firestore/background triggers experimental. Source: https://github.com/firebase/firebase-functions-dart (Apache-2.0).
- Cloud Run functions' own runtime list (Node.js 20/22/24/26-preview, Python, Go, Java, Ruby, PHP, .NET) does not include Dart; Dart works via the Firebase CLI's compiled-binary path. Source: https://docs.cloud.google.com/run/docs/runtimes/function-runtimes

### Cloud Run (container, any language)
- Any language that runs in a container, including a Dart server (Shelf, Dart Frog, Serverpod are the frameworks dart.dev lists; dart.dev also recommends a monorepo with `app`, `functions`, `shared` packages for sharing data classes, serialization and validation). Source: https://dart.dev/server
- Cold start: startup = image streaming + entrypoint + listen on port; mitigations are startup CPU boost, min instances, small startup work. Default max concurrency 80. Image size does not affect startup thanks to streaming. Min instances bill idle time. Source: https://docs.cloud.google.com/run/docs/tips/general
- Firebase tips for the same levers (min instances, global-scope reuse, lazy init): https://firebase.google.com/docs/functions/tips

### Genkit
- Genkit supports Dart, TypeScript, Go and Python. Genkit Dart was announced 2026-03-10 as an early preview; pub.dev now shows `genkit` 1.0.1 (a "Genkit Dart 1.0" post exists; not read). Dart flows can be served via `genkit_shelf`; works as pure Dart client/server and in Flutter; Google, Anthropic, OpenAI providers. Sources: https://genkit.dev/blog/announcing-genkit-dart/ , https://pub.dev/packages/genkit
- Genkit is an AI orchestration layer, not a runtime; it runs inside either option and does not decide the choice.

### Cost
- Cloud Functions on Blaze: 2M invocations/month free then $0.40/M; 400K GB-s and 200K CPU-s free; 5 GB egress free. Spark plan: functions usage listed "Not applicable". Source: https://firebase.google.com/pricing (Dart deployment requires Blaze; assume Blaze for any deploy).
- Cloud Run per-unit prices and free tier: the official pricing page did not render in my fetch, so exact numbers are NOT verified here. Both options bill on the same Cloud Run metering (2nd gen = Cloud Run). At small scale cost is dominated by AI/model calls, not compute; scale-to-zero keeps idle cost near zero, min instances add idle cost.

## Comparison against the ticket's criteria

| Criterion | Functions for Firebase 2nd gen, TypeScript | Cloud Run container, Dart server | Functions for Firebase, Dart (experimental) |
|---|---|---|---|
| Maturity | GA | Dart on Cloud Run works; server framework choice needed | Experimental, may change without notice |
| Cold start | Node cold starts; mitigate with min instances, concurrency up to 1,000 | AOT binary, small, fast (no benchmark found) | AOT; claim only, no figures |
| Cost | Cloud Run metering, free quota, scale to zero | Same metering | Same metering |
| Local emulation | Firebase Emulator Suite for functions, Firestore, Auth etc. | Run the server locally; Firestore/Auth emulators still usable | Emulator supports `onRequest`/`onCall` only |
| Testability | Mature (Node test runners) | Dart test; plain HTTP handlers | Same as Dart, but SDK experimental |
| Deployment | `firebase deploy`, console visibility | Container build + `gcloud run deploy` (or Firebase Hosting rewrite) | `firebase deploy`; not in Firebase console; callable-by-name broken |
| Share model types with Flutter | No direct sharing; use schema + codegen | Yes, shared Dart package | Yes, shared Dart package |
| Ecosystem for MCP, OAuth, AI | Largest (see risks) | Smaller; verify | Smaller; verify |

## Recommendation and rationale

1. TypeScript on Node (22 LTS or 24, whichever the Firebase CLI supports at scaffolding time; the Firebase page still lists only 20 and 22 while Cloud Run functions lists 24) deployed as a 2nd gen HTTPS function. It is GA, has the full emulator, the console visibility, and the most mature Firebase Admin SDK.
2. Keep the domain logic framework-independent behind one HTTP app (e.g. Hono or Express) so the same code runs in a plain Cloud Run container. That keeps the exit to Cloud Run (for long-lived MCP connections, custom concurrency, or Dart) open.
3. Model-type sharing: define the API contract once (OpenAPI or JSON Schema, e.g. generated from zod) and generate Dart client types. This recovers most of the benefit of a shared Dart package without betting v1 on an experimental SDK.
4. Dart backend is the main alternative: choose it only if single-language sharing outweighs maturity. Its blockers today are experimental status, HTTPS/callable-only deploy, experimental Admin SDK, and the callable/console limitations.

## Risks and open items
- Dart Functions may reach GA within the v1 timeline; re-check https://github.com/firebase/firebase-functions-dart before scaffolding.
- Not verified: Cloud Run exact prices; real cold-start benchmarks for Node vs AOT Dart; maturity of a Dart MCP server SDK and Dart OAuth libraries (TypeScript has an official MCP SDK; not re-verified here). The MCP hosting ticket should settle whether MCP runs in the same service or a separate Cloud Run service (long-lived streamable HTTP and OAuth endpoints may favor Cloud Run).
- Node 24 vs Firebase CLI support lag: pick the version the Firebase docs list at scaffolding.
- Deploy requires the Blaze plan (billing account), relevant to the cost-guardrails area.
- Not covered: AI provider choice, database choice, and vector/semantic search store (separate tickets).
