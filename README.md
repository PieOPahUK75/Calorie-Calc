# Daily Tracker — install guide

Five files: `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`. Keep them together in one folder.

This app now has three tools, switchable via the menu icon (☰) in the top-left: the original Calorie Calculator, Press-Ups, and Swimming. All share one install, one Settings→Data backup/restore, and one Cloud sync — everything below still applies to the app as a whole.

## Option A — fastest: Add to Home Screen (no APK)
1. Put the folder somewhere reachable (e.g. a private GitHub Pages repo, Netlify Drop, or your own web host — it must be served over `https://`, not opened as a local file).
2. On your Android phone, open that URL in Chrome.
3. Chrome menu (⋮) → **Add to Home screen** → Install.
4. It now behaves like a native app: own icon, launches full-screen, works offline.

## Option B — real installable .apk via PWABuilder
1. Host the folder the same way as Option A (any `https://` URL works — Netlify Drop at netlify.com/drop is free and takes 30 seconds, no account needed).
2. Go to **pwabuilder.com** on any computer, paste your URL, click **Start**.
3. Click **Package for stores** → **Android** → download the generated `.apk` (or `.aab`).
4. Transfer the `.apk` to your phone, open it. Android will ask to allow install from this source — approve it, then install.

By default your data (weight/fat log, settings) lives only in the app's local storage on your phone — nothing is sent anywhere unless you turn on Cloud sync (below). Use the **Export backup** button in Settings before uninstalling or switching phones, or set up Cloud sync for automatic backup.

## Calculations used
- BMR: Mifflin-St Jeor (`10×kg + 6.25×cm − 5×age + 5`)
- Maintenance (TDEE): BMR × activity multiplier
- Daily calorie deficit: your Settings weekly-loss target (kg/week) × 7,700 kcal ÷ 7, subtracted from TDEE, rounded to nearest 10, never below a 1,200 kcal floor
- Protein target: kg × 1.6
- BMI: kg ÷ (cm²) × 10,000

Core formulas verified against your original spreadsheet's formulas — same inputs produce identical results.

Note: the spreadsheet's "Weekly Change" column summed the *next* 6 days rather than the past 7 (an artifact of how the formula was filled down). The app instead uses a proper trailing average, which is what "weekly change" should mean. The weight-trend/predicted-goal-date figures use a least-squares fit across every logged day (not just first-vs-last), so missed weigh-ins don't skew the projection.

## Press-Ups tracker

Log each set as you do it through the day (e.g. 8, then 5, then 2) — the Today tab shows a running total and lets you delete a mis-entered set. There's no real "midnight rollover" happening in the background (this is a static app with no server); a day's total is just the sum of that calendar day's sets, worked out fresh whenever you look at it. So today's total updates live as you add sets, and the moment the date changes it's automatically yesterday's locked-in total in History — no special handling needed.

Targets are a fixed commitment, not a prediction: in Settings, pick whether you're aiming for a single-day total (e.g. "50 in one day") or a cumulative all-time total (e.g. "10,000 lifetime reps"), set the number, and how many days you're giving yourself — saving converts that into a fixed target date. The History chart plots either daily totals or the cumulative running total (matching whichever target type you picked) against a dashed target line, over the same 7D/30D/1Y/All ranges as the calorie tracker.

## Swimming tracker

Each swim is its own dated session (not a running daily total like Press-Ups), since you can swim more than once a day and might want to backfill a session from an earlier date — the date field on the entry form is editable for exactly that reason.

Per session you log: duration (separate minutes and seconds fields), pool length (metres — pre-filled from your default in Settings, but editable every time, since you might swim a different pool or open water), lengths, distance, and total stroke count. Distance auto-fills as lengths × pool length but stays directly editable, so an open-water swim just needs distance typed in with lengths left blank.

From those totals the app derives:
- **Avg pace** — seconds per 100m (`duration ÷ distance × 100`), the standard swim pace figure.
- **Avg SWOLF** — `(duration ÷ lengths) + (strokes ÷ lengths)`, the standard swim efficiency score (lower is better). This is a session average, not a per-length figure, since only session totals are logged rather than a per-length breakdown.
- **Stroke rate** — strokes per minute (`strokes ÷ (duration ÷ 60)`).

History has the same 7D/30D/1Y/All range picker as the other tools, plus a metric toggle above the chart (Pace / SWOLF / Stroke rate) so you can see whichever trend you care about.

## Cloud sync setup (optional)

The app works fully offline with no account. If you want your data backed up to Google Cloud and kept in sync if you ever use it on a second device, do this one-time setup (free, takes about 10 minutes):

1. Go to **console.firebase.google.com** → **Add project**. Name it anything (e.g. "quick-calorie"). You can decline Google Analytics — not needed.
2. Left sidebar → **Build** → **Authentication** → **Get started** → enable the **Google** sign-in provider (toggle on, pick a support email, save).
3. Left sidebar → **Build** → **Firestore Database** → **Create database** → start in **Production mode** → pick a region near you.
4. Still in Firestore, open the **Rules** tab and replace the contents with:
   ```
   rules_version = '2';
   service cloud.firestore {
     match /databases/{database}/documents {
       match /users/{userId} {
         allow read, write: if request.auth != null && request.auth.uid == userId;
       }
     }
   }
   ```
   This locks every user's data so only that signed-in Google account can ever read or write it. Click **Publish**.
5. Gear icon → **Project settings** → **General** tab → scroll to **Your apps** → click the **</>** (web) icon → register an app (any nickname, skip hosting) → it shows a `firebaseConfig` object with `apiKey`, `authDomain`, `projectId`, etc.
6. Open `index.html` in a text editor, find the block near the top of the `<script>` starting with `const firebaseConfig = {` (search for `PASTE_YOUR`), and replace those placeholder values with the real ones from step 5.
7. Back in Firebase console: **Authentication** → **Settings** tab → **Authorized domains** → **Add domain** → add your GitHub Pages domain (e.g. `yourusername.github.io`), otherwise Google sign-in will fail with an "unauthorized domain" error.
8. Push the updated `index.html` to your repo and reload the app. Settings → Cloud sync now shows a **Sign in with Google** button.

Once signed in, every save pushes to the cloud automatically, and signing in on a second device pulls it back down (whichever device saved most recently wins, so avoid editing on two devices at the exact same moment). Everything still works offline without ever signing in — sign-in is only required to upload/sync.

Sign-in opens as a popup window (not a full-page redirect) — this is intentional and required: redirect-based sign-in doesn't work reliably when the app isn't hosted directly on Firebase (which GitHub Pages/Netlify aren't), since modern browsers block the cross-origin storage access it depends on. If your browser blocks the popup, allow popups for this site and try again.
