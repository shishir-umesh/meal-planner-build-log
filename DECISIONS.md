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
