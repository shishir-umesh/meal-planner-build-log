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

## 8. Household data model and security rules (2026-10-03)

**Context.** Decision 7 put household data in Firestore. Everything a household shares (its settings, members, pantry, recipes and meal plan) needs a shape that the security rules can enforce, that keeps reads within the free plan, and that doesn't block later work on invites, sign-in and per-person privacy.

**Decision.**
- **Scope:** everything lives under one household document. Members are a subcollection, and a user belongs to a household when their member document exists. The rules check that on every read and write, so one household can never see another's data.
- **Private data:** each member's nutrition targets and weight sit in a private document only that member can read. Diet, allergies and dislikes are shared, because the planner needs them for everyone.
- **Pantry:** pantry items hold the current state, and the app listens to them. Every change also writes an inventory event in the same batch. Events are append-only: the rules allow creating them and nothing else, so an undo is a new event. The app reads history on demand and never listens to it.
- **Recipes and meal slots:** a recipe is visible to the household or private to its author, through one field and two scoped queries. A meal slot's id is its date plus its slot, so a slot can't be planned twice.
- **Dates and lookup:** dates are stored as `YYYY-MM-DD` text. Each user has a small document listing their households, capped at 5, so finding them costs one read and never a query.
- **Validation:** the rules check every field's type, allowed values and size. Links must be `http(s)`, ids can't reach into other paths, and every write is stamped with the writer and the server time.
- **Joining:** nobody can join an existing household until invite links exist.
- **Tests:** a shared contract suite runs against an in-memory fake and against the real Firestore code on the emulator. The rules tests run there too, in CI, in about a minute.

**Measured.**
- 49 emulator tests pass: 29 rules tests plus the 20-case repository contract.
- With an allow-everything ruleset, 20 of the rules tests fail, so the tests do catch broken rules.
- Running the contract against Firestore caught one gap: the rules refuse an update to a missing item before Firestore can say "not found". The code now checks first, at one read per pantry change.

**Security.**
- **Done here:** household isolation, owner-only household creation tied to the creator's capped list, member-only private profiles and an append-only event log.
- **Still to do:**
  - Invite links and leave/remove rules (T-021).
  - Sign-in with account linking (T-020). Until then the shipped app can't read production data, which fails closed.
  - App Check and API-key restrictions.
  - Write caps per household.
  - A note that the local caches rely on device encryption.

**Trade-off.**
- **Read cost:** every request costs one extra read for the membership check, and the recipe list needs two listeners.
- **Validation limits:** the rules can't look inside list elements, so the app's own parsers check each document and drop any that don't fit.
- **Owner changes:** the owner is fixed for now. Transferring ownership needs a rule change later.
- **In exchange:** each household's data is isolated by the server, not by trust in the app, and a household of two stays well inside the free quota.

## 9. Nutrition reference data: built from FoodData Central, committed, SQLite on iPhone and JSON on web (2026-10-03)

**Context.** Every nutrition number in the app must come from USDA FoodData Central (FDC), never from a language model, and the app has to work offline. The app needs calories, protein, fiber, carbs and fat per 100 g for thousands of foods, plus household portions such as "1 cup = 158 g". FDC publishes the data as CSV downloads in the public domain (CC0). There are two kinds: Foundation Foods, updated a few times a year, and SR Legacy, the final 2018 release.

**Decision.**
- **Build:** a Python script using only the standard library downloads the two releases, pinned by URL and checksum. It keeps only real foods, which drops about 88,000 lab-sample rows, and writes the per-100 g values and portions for both kinds of food: 8,262 foods and 14,636 portions.
- **Missing values:** a value FDC doesn't give stays empty. It's never zero and never estimated.
- **Energy:** many Foundation foods have no standard energy value, so energy uses FDC's standard value, then "Atwater specific", then "Atwater general". Every number records which FDC field it came from.
- **Commit the generated files** instead of building them in CI or at install time: a 1.8 MB SQLite file and a 2.8 MB JSON copy. The build is byte-for-byte repeatable, and CI checks the committed files without touching the network.
- **iPhone:** reads the SQLite file, which ships inside the app.
- **Web:** reads the JSON from the app's own server. SQLite on the web is still experimental in Expo and would need special security headers on every host.

**Measured.**
- **iPhone:** a release build on the simulator read cooked rice (130 kcal, 2.69 g protein, 0.4 g fiber per 100 g, "1 cup = 158 g") from a file inside the app, with no development server running, in 11 ms.
- **Web:** the browser made requests only to the app's own origin.
- **Tests:**
  - Rice and toor dal match FDC's published values in the build script's tests and in the app's tests.
  - The app's tests run against both committed files.

**Trade-off.**
- **Repository size:** about 4.6 MB of generated data, and a data update is a new commit and a new app build.
- **Two formats:** the web and iPhone copies must never drift apart. The build writes both from the same rows, and both CI and the app's tests check that they match.
- **Choosing a food:** with two kinds of FDC food, the ingredient catalog has to choose which food each ingredient uses.
- **Search:** SQLite full-text search is available on the iPhone but not on the web yet.
