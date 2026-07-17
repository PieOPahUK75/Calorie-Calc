# Quick Calorie — install guide

Five files: `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`. Keep them together in one folder.

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

Your data (weight/fat log, settings) lives only in the app's local storage on your phone — nothing is sent anywhere. Use the **Export backup** button in Settings before uninstalling or switching phones.

## Calculations used
- BMR: Mifflin-St Jeor (`10×kg + 6.25×cm − 5×age + 5`)
- Maintenance (TDEE): BMR × activity multiplier
- Daily calorie target: TDEE × 0.85, rounded to nearest 10 (15% deficit)
- Protein target: kg × 1.6
- BMI: kg ÷ (cm²) × 10,000

All verified against your original spreadsheet's formulas — same inputs produce identical results.

Note: the spreadsheet's "Weekly Change" column summed the *next* 6 days rather than the past 7 (an artifact of how the formula was filled down). The app instead uses a proper trailing average, which is what "weekly change" should mean.
