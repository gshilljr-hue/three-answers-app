# Three Answers to c4 — Android app

Wraps the Queen's Gambit trainer in a real Android app. The whole trainer — board,
chess engine, opening book, analysis — is a single HTML file inside the APK, so the
app works with no signal.

You do **not** need Android Studio, a Mac, or a fast computer. GitHub builds the APK
for you, for free, and hands you a link you can open on the phone.

---

## Build it (the easy way — all in a browser)

**1. Make a repository.** On [github.com](https://github.com), sign in and create a
new repository. Call it anything, e.g. `three-answers-app`. **Private is fine.**

**2. Upload this folder.** On the new repo's page choose **Add file → Upload files**,
then drag in *everything* from this folder — including the hidden `.github` folder,
which is the part that does the building. Commit.

> On a phone the drag-and-drop uploader is fiddly. Easiest phone-only route: install
> the **GitHub** app or use a file manager that can unzip, or just do this one step on
> a computer. Everything after it works fine on a phone.

**3. Turn Actions on.** Open the **Actions** tab. If it asks, click
*I understand my workflows, go ahead and enable them*. The build starts on its own —
give it about 3–5 minutes the first time.

**4. Get the APK.** When the run finishes, go to the repo's **Releases** (right-hand
side of the main page, or `/releases`). There's a release called **latest** with
`three-answers-to-c4.apk` attached.

**5. Install it.** Open that release page *on the phone* and tap the `.apk`. Android
will say it can't install from this source — tap **Settings**, allow it for your
browser, tap back, install. That's the standard sideloading prompt; you only see it
once per browser.

Push a change later and the same `latest` release is updated in place, so the download
link never changes.

---

## Build it on a computer instead

With [Android Studio](https://developer.android.com/studio): **File → Open**, pick this
folder, let it sync, then **Build → Build Bundle(s)/APK(s) → Build APK(s)**.

From a terminal, with a JDK 17+ and the Android SDK installed:

```bash
./gradlew assembleDebug
# -> app/build/outputs/apk/debug/app-debug.apk
```

---

## About the signature

The APK is signed with Android's standard **debug key**. That's deliberate: it's what
lets the cloud build run with no secrets for you to set up, and it installs and runs
exactly like any other app.

Two things to know:

- It's marked debuggable, which costs a little speed. You will not notice it here — the
  chess engine is JavaScript in a WebView either way.
- You can't put a debug-signed APK on the Play Store. For a personal app, that's moot.

**Want a properly signed release build?** Generate a keystore once:

```bash
keytool -genkey -v -keystore release.jks -keyalg RSA -keysize 2048 \
        -validity 10000 -alias trainer
```

Add `signingConfigs` to `app/build.gradle` pointing at it, store the keystore as a
base64 GitHub secret, and swap `assembleDebug` for `assembleRelease` in
`.github/workflows/build-apk.yml`. Keep the keystore file itself out of the repo —
`.gitignore` already blocks `*.jks` and `*.keystore`. Whatever you do, **don't lose
it**: without it you can't ship an update that installs over the old one.

---

## Updating the trainer later

The app is just a shell around one file. To ship a new version of the trainer:

1. Replace `app/src/main/assets/index.html` with the new HTML.
2. Bump `versionCode` (and `versionName`) in `app/build.gradle` — Android refuses to
   install over an existing app unless `versionCode` went up.
3. Commit. The build runs and refreshes the `latest` release.

---

## What's in here

```
app/src/main/
  assets/index.html                  the entire trainer, offline
  java/…/MainActivity.java           ~90 lines: a WebView and a dark-mode bridge
  AndroidManifest.xml
  res/values{,-night}/               colours and themes matched to the page
  res/mipmap-*/                      launcher icons
.github/workflows/build-apk.yml      the cloud build
```

**Permissions:** just `INTERNET`, and only because the page pulls three webfonts from
Google Fonts. Nothing else leaves the device — no analytics, no accounts, no data
collection. Delete that line from `AndroidManifest.xml` for a build with no network
access at all; the page falls back to system fonts and plays identically.

**Requires** Android 7.0 (API 24) or newer.

### A couple of deliberate choices

- **The back button exits the app**, the standard Android behaviour, rather than
  undoing a move. The trainer has its own *Back one move* button, and overloading the
  system gesture would make the app hard to leave. To change it anyway, override
  `onBackPressed()` in `MainActivity` and call into the page's `state.sanHistory`.
- **Dark mode is pushed in from the native side.** A WebView doesn't reliably tell a
  page about the host app's night mode, so `MainActivity` sets the page's own
  `data-theme` attribute from the system setting — which is why dark mode tracks the
  phone exactly instead of half-working.
- **System font scaling is ignored** (`setTextZoom(100)`). The board and the analysis
  panel are laid out against the page's own type scale, and a large accessibility font
  setting would otherwise push the board off-screen.
