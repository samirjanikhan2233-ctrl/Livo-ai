# Livo AI

**An all-in-one AI personal assistant and productivity app for Android.**
Developer / Publisher: **MuhammadSamirKhankabir**

Livo AI lets people chat with a real AI assistant, plan their day, manage tasks, build habits,
set goals, track income and expenses, take notes, get reminders, and earn XP and achievements
along the way — with a free ad-supported tier and a Premium subscription via Google Play
Billing.

> **Read this before you build:** this repository is a complete, real Android Studio project
> (not a mockup). It will **not** run correctly out of the box, because it depends on secrets
> and infrastructure that must be *yours*: a Firebase project, a deployed backend, real AdMob
> ad unit IDs, and real Google Play subscription products. Sections 6–10 below walk through
> setting each of these up. This is intentional and correct security practice — see
> **Security** below for why.

---

## 1. What Livo AI is

Livo AI is a Material 3, Jetpack Compose Android app built around five core areas, reachable
from a bottom navigation bar — **Home, Planner, Habits, Finance, AI** — plus Profile, Settings,
Notes, Goals and Premium reachable from Home/Profile. Every feature described in the product
spec is implemented with real local persistence (Room) and real network architecture
(Retrofit → your backend → AI provider / Google Play), not hard-coded placeholder data.

## 2. Developer

**MuhammadSamirKhankabir** — shown in-app on the About screen and in this README. No other
company identity is implied or invented.

## 3. Technologies

| Layer | Choice |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose, Material 3 |
| DI | Hilt |
| Local database (offline mode) | Room |
| Networking | Retrofit + OkHttp, talking only to Livo AI's own backend |
| Auth | Firebase Authentication |
| Cloud sync target (reference) | Firestore (rules included) or your own SQL DB behind the Node backend |
| Background work / reminders | WorkManager |
| Subscriptions | Google Play Billing Library v7 |
| Ads | Google AdMob SDK |
| Preferences | Jetpack DataStore |
| Backend (reference) | Node.js + Express |
| AI provider | Called only from the backend — provider-agnostic (Anthropic API used as the example) |

## 4. Project structure

```
LivoAI/
├── app/                                  # Android app module
│   ├── build.gradle.kts
│   ├── google-services.json.example      # copy → google-services.json (real Firebase config)
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/muhammadsamirkhankabir/livoai/
│       │   ├── LivoApplication.kt, MainActivity.kt, MainViewModel.kt
│       │   ├── navigation/                # NavGraph, Destinations, bottom-nav wiring
│       │   ├── ui/theme/                  # Material 3 theme, light/dark/system
│       │   ├── ui/components/             # BottomNavBar, BannerAdSlot
│       │   ├── ui/screens/                # splash, onboarding, auth, home, planner, habits,
│       │   │                              # finance, ai, notes, goals, profile, settings,
│       │   │                              # premium, legal — each with its screen + ViewModel
│       │   ├── data/local/                # Room entities, DAOs, AppDatabase, Converters
│       │   ├── data/remote/               # LivoBackendApi (Retrofit) + DTOs
│       │   ├── data/repository/           # Task/Habit/Goal/Note/Finance/Chat/Gamification/Sync
│       │   ├── auth/                      # SessionManager, AuthRepository (Firebase Auth)
│       │   ├── billing/                   # BillingManager (Play Billing v7, server-verified)
│       │   ├── ads/                       # AdManager (centralized AdMob control)
│       │   ├── notifications/             # NotificationHelper, ReminderWorker, ReminderScheduler
│       │   ├── gamification/              # XpManager (levels, achievements)
│       │   ├── di/                        # Hilt modules (Database, Network, App)
│       │   └── util/                      # Constants, NetworkMonitor, SettingsStore, ThemeMode
│       └── res/                           # strings, colors, themes, launcher icon, notif icon
├── backend/                               # Reference Node/Express backend (deploy this yourself)
│   ├── server.js, package.json, .env.example
│   └── firestore/firestore.rules          # example security rules if you use Firestore
├── legal/                                 # Privacy Policy / Terms / Support (Markdown source)
├── play-store/store-listing.md            # Play Store copy + publishing checklist
├── .env.example                           # top-level env var reference
├── keystore.properties.example            # copy → keystore.properties for release signing
├── local.properties.example               # copy → local.properties for local SDK/config
└── README.md                              # this file
```

## 5. Installation (local machine)

**Prerequisites:** Android Studio (Koala/2024.1+), JDK 17, an Android device or emulator on
API 24+.

```bash
git clone <your-repo-url> LivoAI
cd LivoAI
cp local.properties.example local.properties     # then set sdk.dir to your Android SDK path
```

Open the folder in Android Studio and let Gradle sync. At this point the app **will compile**
but auth, AI chat, ads and billing won't fully work until you complete sections 6–10.

## 6. Backend setup

The Android app never talks to an AI provider, Google Play verification API, or a database
directly — it only talks to **your own backend** (`/backend`), which holds all real secrets.

```bash
cd backend
cp .env.example .env      # fill in AI_API_KEY, service account paths, package name, etc.
npm install
npm start                 # runs on http://localhost:8080 by default
```

Deploy it anywhere that runs Node (Cloud Run, Render, Fly.io, a VPS, etc.) behind HTTPS, then
point the Android app at it — see **Environment variables** below (`BACKEND_BASE_URL`).

The backend's endpoints (`/v1/ai/*`, `/v1/billing/*`, `/v1/sync/*`, `/v1/account/delete`) are
fully implemented against the Retrofit contract in `LivoBackendApi.kt`; the two `TODO`s inside
`server.js` are where you plug in your actual database calls (the request/response shapes are
already correct end-to-end).

## 7. Database setup

Two supported paths — pick one:

**Option A — Firestore (fastest to start):**
1. Create a Firebase project, enable Firestore.
2. Deploy `backend/firestore/firestore.rules` (`firebase deploy --only firestore:rules`).
3. Wire `firebase-admin` calls into `server.js`'s `TODO` sections.

**Option B — Your own SQL database:** point `DATABASE_URL` in `backend/.env` at Postgres/MySQL,
and implement the same `TODO` sections with your ORM of choice. The API contract the Android
app expects doesn't change either way.

Every table Livo AI needs is already modeled locally in Room (`data/local/entities`) — mirror
those shapes server-side: users, tasks, habits, habit_completions, goals, milestones, notes,
transactions, budgets, chat_conversations, chat_messages, achievements, user_progress.

## 8. AI setup

1. Get an API key from your AI provider (Anthropic used as the example in `server.js`).
2. Put it in `backend/.env` as `AI_API_KEY` — **never** in the Android app.
3. Optionally swap `callAiProvider()` in `server.js` for a different provider's API.
4. The Android app's `ChatRepository`, `GoalViewModel`, `NoteRepository` and `FinanceViewModel`
   already call the corresponding `/v1/ai/*` endpoints — no client changes needed.

Free-tier vs Premium AI message limits are enforced **server-side** in `checkAndIncrementDailyLimit()` —
the client-side constants in `util/Constants.kt` are only a UX hint.

## 9. AdMob setup

1. Create an AdMob account and app entry; get your real **App ID**, **Banner ad unit ID**, and
   **Rewarded ad unit ID**.
2. Put them in `local.properties` (dev) or your CI secrets (release):
   ```
   ADMOB_APP_ID=ca-app-pub-xxxxxxxxxxxxxxxx~xxxxxxxxxx
   ADMOB_BANNER_ID=ca-app-pub-xxxxxxxxxxxxxxxx/xxxxxxxxxx
   ADMOB_REWARDED_ID=ca-app-pub-xxxxxxxxxxxxxxxx/xxxxxxxxxx
   ```
3. Until you do, the app builds and runs using **Google's official AdMob test IDs**
   (already the default in `app/build.gradle.kts`), so ads are visible and safe to test with
   during development, per AdMob's own testing guidance.
4. All ad logic lives in the single `ads/AdManager.kt` (the required "centralized AdManager").
   Premium users (`BillingManager.isPremiumVerified == true`) never see a banner or rewarded ad —
   this is enforced in `AdManagerHolder` and `BannerAdSlot`, not left to each screen to remember.

## 10. Google Play Billing setup

1. In Play Console, create two auto-renewing subscription products under your app:
   - `livo_ai_premium_monthly`
   - `livo_ai_premium_yearly`
   (these IDs are already referenced in `billing/BillingManager.kt` — rename in both places if
   you change them.)
2. Create a Google Cloud service account with access to the Google Play Developer API, download
   its JSON key, and set `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON_PATH` in `backend/.env`.
3. **Security model (already implemented):** the Android app calls
   `BillingClient.launchBillingFlow`, but a successful Play purchase callback alone never sets
   `isPremium = true`. `BillingManager.verifyAndAcknowledge()` always sends the purchase token to
   the backend's `/v1/billing/verify-purchase`, which calls the real Google Play Developer API
   server-side and only *that* response updates `isPremiumVerified`. This is the "never trust
   `isPremium = true` on the client" requirement, implemented end-to-end.
4. Test with a **license tester** account in Play Console's internal testing track before going
   live — real billing flows require an app uploaded to at least an internal testing track.

## 11. Environment variables

**Android (`local.properties`, git-ignored — see `local.properties.example`):**
```
sdk.dir=/path/to/Android/sdk
BACKEND_BASE_URL=https://api.yourdomain.com/
ADMOB_APP_ID=
ADMOB_BANNER_ID=
ADMOB_REWARDED_ID=
```

**Backend (`backend/.env`, git-ignored — see `backend/.env.example`):**
```
AI_API_KEY=
DATABASE_URL=
AUTH_CONFIG=                # any extra Firebase Admin config you need
ADMOB_APP_ID=
ADMOB_BANNER_ID=
ADMOB_REWARDED_ID=
GOOGLE_PLAY_PACKAGE=com.muhammadsamirkhankabir.livoai
PAYMENT_CONFIG=             # path to your Play Developer API service-account JSON
```

Never commit real `.env`, `local.properties`, `keystore.properties`, or `google-services.json`
files — all are already listed in `.gitignore`.

## 12. APK build (debug — for local testing)

```bash
cd LivoAI
./gradlew assembleDebug
```
Output: `app/build/outputs/apk/debug/app-debug.apk`
Install on a connected device: `./gradlew installDebug`, or copy the APK and `adb install app-debug.apk`.

## 13. AAB build (release — for Google Play)

First complete **Release signing** below, then:
```bash
./gradlew bundleRelease
```
Output: `app/build/outputs/bundle/release/app-release.aab` — this is the file you upload to
Play Console.

If you specifically need a signed release **APK** (e.g. for manual sideload testing) instead of
the AAB:
```bash
./gradlew assembleRelease
```
Output: `app/build/outputs/apk/release/app-release.apk`

## 14. Release signing

```bash
keytool -genkey -v -keystore livoai-release.keystore -alias livoai \
  -keyalg RSA -keysize 2048 -validity 10000
cp keystore.properties.example keystore.properties
# edit keystore.properties with the real storeFile path, storePassword, keyAlias, keyPassword
```
`app/build.gradle.kts` automatically picks up `keystore.properties` if present and signs
`assembleRelease` / `bundleRelease` with it; otherwise release builds fall back to the debug
signing config so the project still builds before you've generated a real keystore (you cannot
upload a debug-signed AAB to Play, so do this before publishing). **Back up your keystore and
passwords somewhere safe** — losing them means you can never update the app on Play again.

## 15. Play Store publishing checklist

See `play-store/store-listing.md` for the full short/full description, screenshots checklist,
Data Safety and Content Rating prep, and category. Summary:

1. Complete Firebase, backend, AdMob, and Billing setup (sections 6–10) with **production**
   values, not test IDs.
2. Host your real Privacy Policy (start from `legal/PRIVACY_POLICY.md`) at a public URL; update
   `PRIVACY_POLICY_URL` in `util/Constants.kt` and in Play Console.
3. Generate a release keystore (section 14) and keep it safe.
4. `./gradlew bundleRelease` → upload the resulting `.aab` to an **internal testing** track
   first.
5. Complete the Data Safety form and Content Rating questionnaire in Play Console truthfully,
   based on your final backend implementation.
6. Test the full flow with real (or license-tester) Google accounts: sign-up, login, every
   feature, a real subscription purchase + restore, and account deletion.
7. Only then promote to production and submit for review.
   **This project has not been published anywhere — publishing is a step you perform yourself
   in Play Console.**

## 16. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Gradle sync fails on `google-services` | You haven't added a real `app/google-services.json` yet — the plugin is applied conditionally, so this should only block Firebase-dependent runtime features, not the sync itself. If sync still fails, double-check the plugin block in `app/build.gradle.kts`. |
| Login/Register does nothing / throws | Firebase Auth isn't configured — add `google-services.json` and enable Email/Password sign-in in the Firebase console. |
| AI chat always errors | Your backend isn't running/reachable, or `AI_API_KEY` isn't set in `backend/.env`, or `BACKEND_BASE_URL` in `local.properties` doesn't point at it. |
| Ads don't show | Using placeholder/invalid AdMob IDs, no internet on the test device, or you're signed in as a Premium (verified) user — Premium is intentionally ad-free. |
| "Purchase could not be verified" | Your backend's Google Play service account isn't configured, or the subscription product IDs in Play Console don't match `MONTHLY_SUB_ID`/`YEARLY_SUB_ID` in `BillingManager.kt`. |
| Release build fails to sign | `keystore.properties` missing or has wrong values — see section 14. |
| App crashes on a normal network error | Please file this as a bug — every repository method here wraps network calls in `try/catch` and surfaces a user-facing error instead of crashing; if you find a gap, it should be fixed, not worked around. |

---

**Free users:** ads on. **Premium users:** ads off (server-verified). **Payments:** Google Play
Billing. **Testing:** build and sideload the debug APK first. **Publishing:** upload the signed
AAB to Google Play yourself when you're ready.
