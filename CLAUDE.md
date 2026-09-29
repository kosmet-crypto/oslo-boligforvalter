# CLAUDE.md – Oslo Boligforvalter

Mobile app for a boligforvalter in an Oslo bydel: flytteprotokoll (innflytting, utflytting),
periodisk kontroll and befaring / tilstandsrapport for municipal flats, with defect follow-up.
Runs as a PWA (GitHub Pages) and as an Android APK (WebView wrapper) from the same code.

## Working with the owner
- The owner writes in Serbian (Cyrillic). Answer in Serbian Cyrillic, short and concrete.
- The app UI, PDF texts and defaults are in Norwegian (bokmål). Code, comments, commits and README in English.
- Propose ideas first when asked for ideas; implement when asked. Ask before large restructures.
- The project is in a test phase: there is no real data to migrate, so old formats need no compatibility code.

## Hard rules
- **No personal data in the repo.** Never put the owner's full name, bydel, addresses or other personal
  details in code, defaults, tests or commit messages. The only credit is "Utviklet av Ivan St."
  (Innstillinger → Om appen, first-start screen and start-screen footer). Everything personal is
  entered by the user in the first-start wizard / Innstillinger.
- **Tenant privacy (GDPR).** The PDF carries the tenant's full name (and phone, if typed). Everything
  that is stored or leaves the app otherwise keeps the tenant only as initials ("Ola Nordmann" → "O. N."):
  - a signed (locked) document is saved through `anonymize()` – initials, no phone, no tenant signature;
  - backups and the Excel/CSV export are always anonymized, also for drafts;
  - the e-mail text for "Del PDF" uses initials.
  Only the tenant is reduced to initials; boligforvalter and the optional extra person keep full names.
  There is no e-mail field for the tenant.
- **Every app of this owner must have a manual "Se etter oppdatering" button** (web and Android),
  plus the quiet check on start. Keep it working when changing anything around updates.
- Keep it offline-capable: no CDNs. Libraries live in `vendor/` (Alpine.js, signature_pad, jsPDF,
  jsPDF-AutoTable). No build step.

## Layout
- `index.html` – all markup and CSS (Alpine.js templates, screens: setup, list/Boliger, bolig, edit,
  avvik, settings, checklist; bottom tab bar).
- `app.js` – all logic: data model, checklists, PDF (`buildPdf`), IndexedDB (`DB`), Alpine component `app()`.
- `sw.js` – service worker for the PWA; `manifest.json`, `icons/`.
- `android/` – WebView wrapper, package `app.boligforvalter`. `MainActivity.java` (bridge
  `window.BoligAndroid`: getVersion, checkForUpdate, saveFile, saveText, shareFile, shareFileText),
  `Ota.java` (silent content updates), `SelfUpdate.java` (APK self-update).
- `.github/workflows/android.yml` – builds the APK on every push to `main` and publishes a release
  (`boligforvalter.apk`, tag `v1.0.<run_number + 10>`).

## Things that are easy to break
- **Silent updates only fetch `index.html` and `app.js`** (`Ota.FILES`). All app code and CSS must stay in
  those two files; a new JS/CSS file would be missing in the APK after a silent update. Changes to
  `vendor/`, icons, `manifest.json` or `android/` need a new APK (the workflow fingerprints them).
- A new bridge method: feature-detect it in JS (`if (BoligAndroid.x)`) so older APKs keep working. Only if
  the page cannot work without it, bump `Ota.NATIVE_API` and `<meta name="app-native">` together.
- On every web change bump `WEB_VERSION` in `app.js` (the web update check compares it with the published
  `app.js`) and `VERSION` in `sw.js`.
- `versionCode` = run number + 10 (build.gradle and the workflow tag must match). The signing key is
  committed on purpose so every build can update the installed app.
- Signature pads may be created while hidden (width 0); they are sized on first touch and on resize.
  jsPDF built-in fonts are Latin-1 only – pass all PDF text through `pdfText()`.
- Rooms in a signed document sit inside `<fieldset disabled>`; room headers are divs so they still open.

## Data model (short)
Report: `type` (Innflytting | Utflytting | Periodisk kontroll | Befaring), `adresse`, `leilighet`, `dato`,
`leietaker {navn, telefon, tilstede}`, `ekstra {navn, rolle}`, `keys[]`, `strom`, `malerFoto[]`, `rooms[]`
(items with status OK/FEIL, options, selected, fagperson, hast, kostnad, belastes Utleier|Leietaker|Kjent,
photos, tiltak {status Åpen|Bestilt|Utført, bestilt, utfort}), `signatures {forvalter, leietaker, ekstra}`,
`locked`, `anon`, `korrigerer`, `compareWith`. Settings hold profile, logo, addresses, editable checklist,
price list, texts and e-mail templates (`PROFILE_KEYS` travel in profile/backup files, never the PIN).

## Testing
No test suite in the repo. Test in a real browser with Playwright (installed globally in the container):
serve the folder (`npx http-server -p 8765 -s -c-1 .`), drive the UI with
`require('/opt/node22/lib/node_modules/playwright')`, and check generated PDFs (e.g. with PyMuPDF).
Cover at least: first-start wizard, new innflytting → sign → PDF (full name in PDF, initials in IndexedDB),
start utflytting (keys/prices/claim/deadline/comparison), backup and CSV contain no full names,
avvik tab, checklist editor. The Android part cannot be built locally (no SDK); CI builds it – check the
Actions run after pushing.
