# Decisions

Short records of the choices that shape the build: the context, the decision, and what it costs us. Newest at the bottom.

## 1. Household first, with gates (2026-09-28)

**Context.** Every feature this app needs exists in some other app, and two well-known meal apps have shut down (Yummly in 2024, Mealime in 2026). The business case is unproven.

**Decision.** Build for our own household first. Move to a friends beta only after we use it 4 weeks in a row, and to a public launch only if 5+ other households keep planning with it after 30 days.

**Trade-off.** Slower path to users, but no time spent polishing something we wouldn't use ourselves.

## 2. Local-first, zero running cost (2026-09-28)

**Context.** A side project with no revenue should cost nothing to run.

**Decision.** Everything runs on the phone: the database, the planner, receipt OCR and an optional LLM. The only cloud piece is Firebase's free plan, to sync household phones. Reminders are notifications scheduled on the phone, and calendar blocks are written to the phone's own calendar, so there is no push server and no calendar API review.

**Trade-off.** Free quotas cap growth; a public launch needs a paid tier to cover sync. Features that need a server (Instacart handoff, importing emailed receipts) wait or need a small free proxy.

## 3. Rule-based planner; AI is optional and on-device (2026-09-28)

**Context.** Reviews of AI meal planners complain about repetitive meals and inconsistent nutrition numbers.

**Decision.** A rule-based scoring engine plans the week: hard filters for diets, allergies and time, then weighted scores for expiring ingredients, nutrition targets, variety and ratings. An on-device LLM (Apple Foundation Models, Gemini Nano) only helps with receipt matching and free-text edits, on phones that support it.

**Trade-off.** Less "magic". Every AI step needs a non-AI fallback, which also covers older phones.

## 4. Approximate inventory (2026-09-28)

**Context.** Manual entry is the top complaint in pantry-app reviews, and it is why people stop using them.

**Decision.** Track exact quantities only for perishables and proteins; staples like spices, dals and flours are have, low or out. Checking off the shopping list and marking a meal cooked update the pantry automatically.

**Trade-off.** Less precise stock counts in exchange for an inventory we'll actually keep up to date.

## 5. Expo for iOS, Android and web (2026-09-28)

**Context.** One part-time builder, and phones on both platforms are possible.

**Decision.** Expo (React Native, TypeScript) with a development build.

**Trade-off.** On-device OCR and LLM access need small native modules, written with the Expo Modules API.

## 6. iOS and web only; Apple Developer Program now (2026-10-03)

**Context.** Both phones in our household are iPhones: an iPhone 17 Pro Max and an iPhone 14 Pro Max, both on iOS 26 today and moving to iOS 27. Development builds on an iPhone need Apple code signing. Free provisioning expires every 7 days and is limited to 3 devices, so the app on my wife's phone would stop opening every week.

**Decision.** v1 supports iOS and web only, which narrows decision 5; no Android in v1. Development builds go on both iPhones, and I joined the Apple Developer Program ($99/yr) now rather than at the beta. The 14 Pro Max has no Apple Intelligence, so it is the baseline device.

**Trade-off.** $99 a year before we know the app is worth it, the only running cost so far. Android friends get the web build until Android is added.

## 7. Household data in Firestore: native SDK on iPhone, JS SDK on web (2026-10-03)

**Context.** Pantry, today's plan, the shopping list and cook mode must work with no signal, and a change on one phone must show on the other within 2 seconds. Everything has to fit Firebase's free Spark plan (50K reads, 20K writes a day) with no server of our own, on iPhone (development build) and the web. A 4-hour spike compared three options and prototyped the leading one.

**Options.**
- **A. Firebase JS SDK on both.** Least setup, but under React Native the JS SDK has no IndexedDB, so it falls back to a memory cache (firebase-js-sdk issue #7947, open). On the iPhone that means writes made offline are lost if the app is closed, and every cold start re-reads, and pays for, everything the app listens to. Fine on the web.
- **B. React Native Firebase on iPhone, the full JS SDK on web, behind one repository interface.** The native iOS SDK keeps a persistent offline cache and write queue by default. The web uses the JS SDK with its IndexedDB cache. Both expose the same modular API, so one repository implementation is shared and only a small SDK file differs per platform. Costs a native rebuild and config plugins.
- **C. SQLite on the phone as the source of truth, with our own sync to Firestore.** Full control and the fewest reads, but we would write the outbox, the listeners and per-field conflict handling ourselves. No free library syncs SQLite with Firestore: RxDB's production SQLite storage is paid, and PowerSync, Zero and WatermelonDB need a server. expo-sqlite on the web is still alpha. The most effort to build and to maintain.

**Decision.** Option B. Household data lives in Firestore, read and written through a repository interface in `src/data`: React Native Firebase on iOS and the full Firebase JS SDK, with a persistent IndexedDB cache, on the web. Reference data (FoodData Central, FoodKeeper) stays a bundled read-only SQLite file. This refines decision 2: the phone's local copy of household data is Firestore's offline cache, not our own database.

**Measured.**
- Each test added an item on one device. The other device listened for it and wrote an acknowledgement, and the sender timed the round trip on its own clock. A round trip under 2 seconds means each one-way sync is well under 2 seconds.
- **Round trips:**
  - Web → iPhone simulator → web: median 312 ms, p95 407 ms, max 407 ms (10 items).
  - Simulator → web → simulator: median 318 ms, p95 345 ms, max 345 ms (10 items).
  - iPhone 17 Pro Max → web → iPhone, over Wi-Fi: median 377 ms, p95 584 ms, max 584 ms (5 items).
  - Web → iPhone → web: 340–462 ms (5 items).
- **Offline:**
  - The iPhone was put in Airplane mode with Wi-Fi off and two items were added; both reached the web after reconnecting.
  - On the simulator, an item was added offline and then the app was killed. It never reached the server while the app was down, and it synced on relaunch, so the native write queue survives a restart.
  - On the web, an item added with the network cut synced on reconnect, and a pending write survived a page reload, held in IndexedDB.
- **Reads:**
  - Reopening the iPhone app with a 22-item list billed 2 reads, because the persistent cache resumed the listener.
  - Each new item costs about one read per listening device, plus about one more for its acknowledgement in the test.
  - A two-person household on a typical day is estimated at about 100 writes and 9K reads. The reads are mostly full re-reads of about 430 documents when the app opens after more than 30 minutes away. That is under 1% of the free writes and about 20% of the free reads.
- **Finding:** React Native Firebase's own web fallback is Firestore Lite, with no listeners and no offline cache. That is why the web uses the JS SDK directly.

**Security.** The spike's rules keep each household's data to its members. Firebase config stays in gitignored files, with an empty `.env.example` committed; a review confirmed none of it was ever committed. The review also found what the real design (T-008 onwards) must add:
- Single-use, expiring invite tokens from a secure random generator, instead of a short shared code.
- Owner, leave and remove-member rules, plus export and delete.
- App Check (App Attest on iOS, reCAPTCHA on web), and API keys restricted to the app and domains.
- Per-member private data for weight and eating logs.
- Linking anonymous sign-in to Sign in with Apple or email, so a reinstall doesn't lose access.
- Caps on writes per household.
- A note that the local caches are not encrypted beyond the device's own encryption.

**Trade-off.**
- Two SDKs to keep in step, a native rebuild for every Firebase upgrade, and dependence on Firestore's offline cache, which gives last-write-wins per field rather than our own conflict rules.
- The append-only event log has to be designed so listeners read current state, not the whole history, or reads will grow over time.
- The web keeps household data in IndexedDB.
- In exchange, offline and real-time sync come built in and free, without writing a sync engine.
