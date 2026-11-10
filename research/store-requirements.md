# App Store and Google Play requirements for Vestis v1

Research for issue #11. Researched 2026-11-10 against primary sources only (Apple, Google, EU legal texts). Not legal advice; items marked **[legal review]** are judgment calls a lawyer or DPO should confirm.

Vestis v1 assumptions: free, public, multi-user, Flutter + Firebase, accounts, body photos for the Avatar, AI-generated images (Virtual try-on of real people), third-party AI providers, an MCP server with per-user OAuth, UI in German and English, EU users in scope.

## Source versions

| Source | Version seen | URL |
|---|---|---|
| Apple App Review Guidelines | "Last Updated: October 16, 2026" | https://developer.apple.com/app-store/review/guidelines/ |
| Apple upcoming requirements | Xcode 26 / iOS 26 SDK since 2026-04-28; min deployment target iOS 13 since 2026-09-09; new age ratings since 2026-01-31; DSA trader status | https://developer.apple.com/news/upcoming-requirements/ |
| Apple age ratings | current page (no date) | https://developer.apple.com/help/app-store-connect/reference/age-ratings/ |
| Apple app privacy details | current page (no date) | https://developer.apple.com/app-store/app-privacy-details/ |
| Google Play account deletion | https://support.google.com/googleplay/android-developer/answer/13327111 | |
| Google Play AI-Generated Content policy | page undated (c 2026) | https://support.google.com/googleplay/android-developer/answer/14094294 |
| Google Play UGC policy | undated | https://support.google.com/googleplay/android-developer/answer/9876937 |
| Google Play Data safety form | undated | https://support.google.com/googleplay/android-developer/answer/10787469 |
| Google Play content rating (IARC) | undated | https://support.google.com/googleplay/android-developer/answer/9859655 |
| Google Play target API level | Android 16 / API 36 by 2026-08-31 (extension to 2026-11-01) | https://support.google.com/googleplay/android-developer/answer/11926878 |
| Google Play new personal accounts testing | https://support.google.com/googleplay/android-developer/answer/14151465 | |
| GDPR, Regulation (EU) 2016/679 | EUR-Lex | https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679 |
| AI Act Art. 50 | Regulation (EU) 2024/1689 as amended by Regulation (EU) 2026/1744 ("Digital Omnibus on AI", PE-CONS 30/1/26 REV 1) | https://data.consilium.europa.eu/doc/document/PE-30-2026-REV-1/en/pdf |

## Short answer

Both stores accept a free multi-user app with body photos and AI images, provided Vestis (1) offers in-app account deletion plus a web deletion link, (2) publishes a privacy policy that names its AI providers, (3) obtains explicit consent before sending photos to third-party AI, (4) ships reporting/moderation safeguards for AI-generated output, (5) fills in the privacy label and Data safety form truthfully including AI-provider flows, and (6) handles GDPR obligations (DPIA, processor contracts, transfers, special-category risk for body photos, minors). No store rule forces a paid or subscription structure, and no store bans AI avatars of the user's own body.

## Apple App Store

### Account creation and deletion
- 5.1.1(v): "If your app supports account creation, you must also offer account deletion within the app." Deletion must be in-app (not just a support email). Revocable social login is also required, and an app "may not store credentials or tokens to social networks off of the device."
- 5.1.1(v) also says apps without significant account-based features should work without login. Vestis is account-based by nature (wardrobe sync, backend domain logic), so a login is justified; explain this in review notes.
- 2.1(a): provide a demo account (and keep the backend running) in App Review notes, with a pre-populated Wardrobe so the reviewer can see Virtual try-on.

### Sign in with Apple (4.8 Login Services)
- 4.8 applies only if a third-party or social login (Google, Facebook, etc.) sets up or authenticates the primary account. Then another login service must be offered that limits collection to name and email, allows hiding the email, and does not track for advertising without consent. Sign in with Apple satisfies this.
- Exempt: apps that "exclusively use your company's own account setup and sign-in systems." Firebase email+password (own account system) alone is exempt.
- Implication: if v1 offers "Sign in with Google", it must also offer Sign in with Apple (or an equivalent privacy-preserving option). If v1 offers only email/password or email link, 4.8 does not apply. Sign in with Apple also requires token revocation on account deletion (Apple's account-deletion flow for Apple-authenticated accounts; confirm in Apple's Sign in with Apple docs when implementing).
- MCP per-user OAuth is a separate login for a third-party client (the user authorises an AI client against the Vestis backend); it authenticates against Vestis's own accounts, so it does not itself trigger 4.8.

### Privacy policy and labels
- 5.1.1(i): a privacy policy link is mandatory in App Store Connect metadata and inside the app. It must list data collected, how, all uses, that third parties give equal protection, retention/deletion policy, and how to revoke consent or request deletion.
- 5.1.1(ii): consent for collection, with an easily accessible way to withdraw it. 5.1.1(iii): data minimisation, prefer the out-of-process picker (PHPicker / share sheet) over full Photos library access. Vestis should use the system photo picker and camera permission only when the user triggers capture.
- 5.1.2(i): "You must clearly disclose where personal data will be shared with third parties, including with third-party AI, and obtain explicit permission before doing so." This is the key rule for body photos, avatars and product-photo import sent to an AI provider: show a dedicated in-app consent screen naming the provider(s), before the first upload, and allow withdrawal.
- 5.1.2(ii): data collected for one purpose may not be repurposed without further consent. Do not use body photos to train models or for analytics.
- App Privacy Details (nutrition label): disclose per data type, including third-party partner code. Relevant types: Contact Info (email, name), User Content (Photos or Videos, Other User Content), Identifiers (User ID), Usage Data, Diagnostics, and possibly Sensitive Info / Health & Fitness if body measurements are stored (judgement **[legal review]**). "Collect" means transmitted off-device and kept longer than needed to service the request in real time. Data sent to the AI provider for a single request and discarded is not "collected" for the label, but photos stored in Vestis storage are collected and linked to the user. Answers can be edited without an app update. Privacy Policy URL required.
- Tracking: if no ad SDKs or data-broker sharing, declare "Data Not Used to Track You"; no ATT prompt is needed.

### AI and real-people imagery
- No dedicated generative-AI section in the guidelines as of 2026-10-16.
- 1.2 User-Generated Content explicitly covers "AI content creation apps". Required safeguards: filtering objectionable material, a mechanism to report offensive content with timely responses, the ability to block abusive users, and published contact info. In Vestis, generated images are private to the user (no sharing in v1), so blocking is moot, but filtering at the provider level, an in-app "report this result" action, and visible contact info are cheap and expected. If sharing of outfits/avatars between users is ever added, full UGC rules apply.
- 1.2 also bars apps for "objectification of real people". 1.1.4: no sexual or pornographic material. Virtual try-on must not allow nudity generation or "undress" results: constrain prompts and use the provider's safety filters; reject uploads that are not of the user.
- 1.2.1(a) / 4.7: not applicable (no creator content or mini apps in v1). Note 4.7 covers "chatbots"; the MCP server is a remote backend that third-party AI clients connect to, not software hosted inside the app, so it is not a 4.7 mini app.
- 5.1.1(viii): apps may not compile personal information about others without consent. Avatar must be of the account holder only (see risks).
- 1.4.1 medical: body-measurement features must not claim health or diagnostic accuracy.

### Age rating
- Since 2026-01-31 Apple uses 4+, 9+, 13+, 16+, 18+ and requires developers to have answered the updated questionnaire. Answers must be honest (2.3.6).
- Questionnaire areas: In-App Controls, Capabilities (UGC, social media, messaging, unrestricted web access, advertising), Mature Themes, Medical or Wellness, Sexuality or Nudity, Violence, Chance. Expect 4+ if there is no social feed, no free web browsing and no mature themes. A product-link import that opens arbitrary web pages in an embedded browser would count as unrestricted web access (16+); use link-fetching on the backend or the system browser instead. Fashion with swimwear or underwear try-on could trigger the "suggestive themes" answer; decide the policy before answering.
- Apple does not require age verification for 4+ apps, but GDPR minors rules still apply (below).

### Other submission mechanics
- Build with Xcode 26+ / iOS 26 SDK (since 2026-04-28); deployment target iOS 13 or later (since 2026-09-09).
- EU DSA trader status required for distribution in the EU App Store (since 2025-02-17). A free app run by an individual still must declare trader or non-trader status; publishing a public product as a business activity probably means trader, with a public address and contact **[legal review]**.
- 2.3: accurate metadata and real screenshots; no placeholder content. Use fictional account data in screenshots (2.3.9): do not show real people's body photos in marketing.
- Notes for Review should describe the AI features and the MCP server (2.3.1(a): generic descriptions are rejected).

## Google Play

### Account and developer prerequisites
- New personal developer accounts (created after 2023-11-13) must run a closed test with at least 12 testers opted in for 14 continuous days before applying for production access; review usually under 7 days. Organisation accounts are not subject to this. Plan lead time of roughly three weeks.
- Target API level: new apps and updates must target Android 16 (API 36) by 2026-08-31 (extension to 2026-11-01 now expired as of today's date). Flutter's stable Android target must be API 36.
- Developer must complete the EU DSA trader declaration in Play Console (the dedicated help URL I tried returned 404, so exact wording was not verified; check in Play Console).

### Account deletion
- User Data policy: apps that let users create accounts must provide a prominent in-app way to request deletion of the account and associated data, plus a working web link where deletion can be requested without reinstalling. The web page must reference the app or developer name. Data kept for security, fraud prevention or compliance must be disclosed. Developers should delete data on their own servers and ask service providers (e.g. the AI provider) to delete too. Answer the Data deletion questions in the Data safety form; the store listing shows a data deletion badge.
- Partial deletion (e.g. only deleting some data types) is allowed but must be explained; for Vestis, delete the whole account and all pieces, photos, avatars and generated images.

### Privacy policy and Data safety form
- Privacy policy: active public URL, valid SSL, no geo restrictions or login, standard web page (not PDF), specific to the app; link both on the listing and inside the app.
- Data safety form is mandatory for every published app, even those collecting no data. Declare data transmitted off device including by third-party SDKs and webviews, per data type: required or optional, purpose (app functionality, account management, etc.), shared or not, encrypted in transit, deletion available. Ephemeral processing (in memory only, discarded after the request) need not appear on the listing but belongs in form answers. Transfers to service providers acting on the developer's behalf are not "sharing" under the form's exceptions; sending photos to a third party that uses them for its own purposes would be.
- Declarations must be accurate; discrepancies can lead to enforcement. Include Firebase SDK data (Auth, Analytics, Crashlytics, Messaging) per each SDK's published guidance.

### AI-Generated Content policy
- Applies to apps where image-generation is a feature ("text-, voice-, or image-to-image apps that generate images"), which includes Virtual try-on. Developers are responsible for ensuring generative AI apps do not generate offensive content (violating Inappropriate Content, child exploitation or deceptive behaviour policies).
- Examples prohibited: non-consensual deepfake sexual material; recordings of real people used for scams; content encouraging harm; deceptive election content; bullying/harassment content; apps primarily sexually gratifying.
- The page lists no explicit in-app flagging requirement, but Google's published guidance elsewhere (not verified here) expects an in-app way to report offensive generated content. Treat a report action as required to be safe. The same reporting feature serves Apple 1.2.
- Google's UGC policy applies where content is visible to other users; with v1 private generated images, UGC does not apply, but terms of use + reporting are required if sharing ships.

### Content rating
- Every app needs an IARC content rating questionnaire in Play Console (PEGI in Europe, USK in Germany, ESRB in the Americas, etc.). Misrepresentation can lead to removal. The page does not describe generative-AI questions; answer according to the actual output possibilities.

### Permissions
- Camera, and photo access via the Android photo picker where possible, to minimise declared sensitive permissions. Use prominent in-app disclosure before sending photos to a third party.

## GDPR and EU law (users in the EU)

Vestis is established where its operator is; either way, offering the service to people in the EU brings GDPR into scope (Art. 3(2) if the operator is outside the EU, which also requires an EU representative under Art. 27).

- **Lawful basis and consent (Art. 6, 7, 9).** Contract (Art. 6(1)(b)) covers the core wardrobe features. For body photos sent to AI providers, rely on explicit consent, which is also what Apple 5.1.2(i) demands. Art. 9(1) prohibits processing special-category data, including biometric data "for the purpose of uniquely identifying", and data concerning health, unless an Art. 9(2) exception applies, with explicit consent in 9(2)(a). Recital 51: photographs "should not systematically be considered to be processing of special categories", only when processed through specific technical means allowing unique identification. A body photo used only to render clothing is probably not biometric data, but a body model with measurements might reveal health data and a face in the photo can be identifying **[legal review]**. Safest design: collect explicit Art. 9(2)(a) consent anyway, and avoid face recognition or identity matching.
- **DPIA (Art. 35).** Required where processing "is likely to result in a high risk". Large-scale processing of body images, new technology (generative AI) and possibly minors make a DPIA likely advisable; document one before launch.
- **Processors and transfers (Art. 28, Chapter V, Art. 44).** Firebase/Google Cloud and every AI provider are processors: sign DPAs, choose EU regions where feasible (Firestore, Cloud Storage, Functions), and rely on adequacy or SCCs for transfers to the US. Verify each AI provider's retention and training-use terms (opt-out of training; zero retention endpoints).
- **Data subject rights (Art. 12 to 22).** Access, portability, rectification, and erasure (Art. 17(1)). In-app deletion must also remove generated images, avatars and backups within a defined window, and cascade to AI providers. Provide data export.
- **Transparency (Art. 13).** The privacy policy must name controller contact, purposes, bases, recipients (AI providers), transfers, retention, and rights. Provide it in German and English, matching the UI.
- **Children (Art. 8).** Where consent is the basis for an information society service offered to a child, the child must be at least 16, unless a Member State lowers it (not below 13; Germany keeps 16). Simple options: terms of use set a minimum age of 16 and ask for confirmation; do not target minors; no age-gating beyond declaration is required by the stores for 4+ apps. Body photos of minors are a heightened risk, so the 16+ floor is advisable.
- **Breach notification (Art. 33).** 72 hours to the supervisory authority; prepare a procedure.
- **DPO (Art. 37).** Probably not mandatory for a small operator but body-image processing at large scale could trigger it **[legal review]**.
- **Imprint.** If the operator is in Germany, a TMG/DDG-style imprint (Impressum) is needed in-app/on the website; this is German national law and was not verified here.

### EU AI Act (Regulation (EU) 2024/1689 as amended by Regulation (EU) 2026/1744)
- Art. 50(2): providers of AI systems generating synthetic image content must mark outputs in a machine-readable, detectable format. Art. 50(4): deployers of systems generating image content resembling real persons that could falsely appear authentic must disclose that it is artificially generated. Applies from 2026-08-02.
- Digital Omnibus on AI (PE-CONS 30/1/26 REV 1, 8 July 2026; Regulation 2026/1744 per secondary reports of publication on 2026-07-24): providers of generative systems placed on the market before 2026-08-02 must comply with Art. 50(2) by 2026-12-02. A system first placed after 2026-08-02 must comply immediately.
- Vestis mostly deploys third-party models (Vestis = deployer, the model vendor = provider), so the vendor's watermarking (e.g. C2PA, SynthID) should carry through; Vestis should label try-on renderings as AI-generated in the UI and not strip metadata. If Vestis fine-tunes or wraps a model under its own name it could be treated as a provider **[legal review]**.
- New prohibitions from 2026-12-02 (Art. 5(1)(ba),(bb)): AI systems generating realistic intimate or sexually explicit images of an identifiable person without explicit consent, and CSAM-type material. Virtual try-on with real body photos is exactly the abuse surface; implement nudity/sexual-content blocking and refuse photos that are not of the account holder to the extent technically possible. This aligns with the store rules above.

## Hard-requirements checklist for v1

Apple
- [ ] In-app account deletion (incl. Sign in with Apple token revocation if used) (5.1.1(v)).
- [ ] If any third-party login (e.g. Google) is offered, also offer Sign in with Apple or equivalent (4.8); email/password only is exempt.
- [ ] Privacy policy URL in App Store Connect and in-app (5.1.1(i)).
- [ ] In-app explicit consent naming third-party AI providers before sending any personal data (5.1.2(i)); withdrawal path (5.1.1(ii)).
- [ ] Use PHPicker/camera on demand rather than full Photos access (5.1.1(iii)).
- [ ] AI content creation safeguards: filter, report mechanism, contact info (1.2); no sexual or "undressing" output (1.1.4).
- [ ] Privacy nutrition label, including third-party AI and Firebase SDKs.
- [ ] Age rating questionnaire answered honestly (2.3.6); avoid embedded open web browsing.
- [ ] Demo account with pre-populated data; backend live; detailed Review Notes explaining AI features and MCP (2.1(a), 2.3.1(a)).
- [ ] Xcode 26+, iOS 26 SDK, deployment target iOS 13+ (upcoming-requirements page).
- [ ] DSA trader status declared for EU distribution.
- [ ] Screenshots with fictional data only (2.3.9).

Google Play
- [ ] In-app account deletion path plus public web deletion URL; Data safety deletion answers.
- [ ] Privacy policy at public URL, linked in-app and on the listing.
- [ ] Data safety form completed incl. SDKs and AI flows; accurate.
- [ ] Prominent in-app disclosure/consent before sending photos to third-party AI.
- [ ] AI-Generated Content policy compliance: safety filtering of image outputs, no deepfake sexual content, in-app report action.
- [ ] IARC content rating questionnaire.
- [ ] Target API 36 (Android 16).
- [ ] If personal developer account: closed test with 12+ testers for 14 days; or use an organisation account.
- [ ] DSA trader declaration in Play Console.

GDPR / EU
- [ ] Privacy policy (de, en) with controller identity, purposes, bases, recipients, transfers, retention, rights.
- [ ] Explicit consent for body photos / AI processing; consent withdrawal.
- [ ] DPIA documented before launch; DPAs with Google/Firebase and every AI provider; transfer mechanism; no provider training on user data.
- [ ] Erasure cascade (storage, derived images, backups, AI-provider copies) and data export.
- [ ] Minimum age 16 in terms; no marketing to minors.
- [ ] AI-generated labelling of try-on images, preserve provider provenance metadata (AI Act Art. 50).
- [ ] Block sexual/intimate generation; verify only the account holder's photos are used (AI Act Art. 5(1)(ba) from 2026-12-02).
- [ ] Breach procedure (72 h); EU representative if operator is outside the EU; imprint if in Germany.

## Open risks and unknowns

1. **Photos of other people.** The stores cannot technically verify that a body photo shows the account holder. Terms, a declaration at upload, provider-side filters and a report flow are the practical mitigation. Consider liveness or selfie matching only if willing to take on biometric data (Art. 9) and DPIA consequences; the recommendation is not to.
2. **Special-category status of body photos** is unsettled for each design (measurements, face visible). Needs a decision and possibly legal review.
3. **AI provider terms are not yet chosen.** Retention, training use, EU residency and Apple's "equal protection" statement (5.1.1(i)) depend on the provider; this research ticket cannot settle it. The Data safety form and privacy label wording depend on it.
4. **Google AI-Generated Content policy page** carries no dated changelog and does not state an in-app flagging requirement; Google may tighten it. Re-check at submission.
5. **Review rejection risk is highest** for Apple 5.1.2(i) (consent wording naming AI providers) and for missing demo data. A first-submission rejection is likely if the consent screen is generic.
6. **AI Act status**: the omnibus text was verified from the Council PE-CONS document (8 July 2026); publication in the Official Journal on 2026-07-24 comes from secondary reports and was not checked on EUR-Lex.
7. **Sign in with Google** convenience versus the extra Sign in with Apple work (token revocation on deletion). Decision for the map.
8. **Personal vs organisation developer accounts** affect Play launch lead time (closed test) and DSA trader exposure (public address). Decision for the map.
9. **MCP server** is not mentioned in store policies; the third-party AI clients that connect via OAuth are outside the store review, but data they pull (piece photos, avatar images) leaves the app. The consent and privacy policy should say that the user chooses to grant an AI client access, and avatar images should perhaps be excluded from the MCP surface by default.
10. **German-specific obligations** (Impressum, age 16 under national law, consumer-law notices) were not researched.

## Not verified

- Exact Apple wording on Sign in with Apple token revocation on deletion (Apple developer documentation, not reviewed).
- Google's Play Console DSA trader page (404) and Google's separate guidance on in-app reporting for generative AI.
- Anything beyond the first ~100k characters of the GDPR page was only partially read; Arts. 22, 25 were not quoted.
