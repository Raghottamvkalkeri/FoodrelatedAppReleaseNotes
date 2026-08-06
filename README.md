# Food Related App — Release Runbook

Internal procedure for cutting a production release of the **Food Related** app (React Native, iOS + Android). Covers flipping the environment config from `devFR` to `prodFR`, swapping signing/config files, bumping version numbers, and pushing the build through Xcode and the Google Play Console.

> Also available as a browsable site: open [`index.html`](./index.html) in a browser, or serve the folder as static site — the screenshots referenced below live in [`assets/`](./assets).

---

## Contents

1. [Open the project](#1-open-the-project)
2. [Switch the environment: `devFR` → `prodFR`](#2-switch-the-environment-devfr--prodfr)
3. [Swap the GoogleService-Info file](#3-swap-the-googleservice-info-file)
4. [Bump the Android version (`build.gradle`)](#4-bump-the-android-version-buildgradle)
5. [Point at the release keystore](#5-point-at-the-release-keystore)
6. [Xcode: build, archive, distribute](#6-xcode-build-archive-distribute)
7. [Android: build & publish to Play Console](#7-android-build--publish-to-play-console)
8. [Reference: identifiers & naming conventions](#8-reference-identifiers--naming-conventions)

---

## 1. Open the project

Open the project in the code editor you're most comfortable with (this runbook uses VS Code). Inside the `app` folder, go to `Utils/utils.js` — this is where the environment switch for the release lives.

App folder shape, for reference:

```
app
├─ API
│  ├─ analyticsErrorLogger.js
│  ├─ errorHandler.js
│  ├─ networkAPI.js
│  └─ paths.js
├─ Analytics
├─ AppRoutes
│  ├─ AppRouter
│  ├─ GuestRouter
│  └─ rootRouter.js
├─ Assets
│  ├─ fonts
│  ├─ icons
│  └─ images
├─ Constants
├─ Modules
│  ├─ Common
│  ├─ Components
│  ├─ Screens
│  └─ Views
├─ ReduxSrore
│  ├─ sagas
│  ├─ slices
│  └─ store.js
├─ Static
│  └─ Images
└─ Utils
   ├─ functions.js
   ├─ helpers.js
   └─ utils.js   ← environment config lives here
```

---

## 2. Switch the environment: `devFR` → `prodFR`

`getKeyValueUrl()` in `utils.js` switches on an environment string to decide which API URL, keys, and app version get baked into the build.

```js
const getKeyValueUrl = () => {
  let key, url, value, version, heroku, gSignInWCID;

  // change 'devFR' to 'prodFR' to build for production
  switch ('prodFR') {

    case 'prodFR':
      url     = 'https://www.foodrelated.com/';
      value   = "dummykeyvalue";
      key     = "dummykey";
      // bump the version — e.g. 1.4.5 → 1.4.6
      version = '1.4.6';
      heroku  = 'https://curbsy.herokuapp.com/reserve/FoodRelatedCurbside/1669183200/13/';
      gSignInWCID = '265662000436-jgc9nqbp7v565l6qt2c8gdfk6l7b4qsv.apps.googleusercontent.com';
      break;
  }

  return { key, url, value, version, heroku, gSignInWCID };
};
```

**Checklist**
- [ ] Switch statement input changed from `'devFR'` to `'prodFR'`
- [ ] Inside the `prodFR` case, `version` incremented (e.g. `1.4.5` → `1.4.6`)
- [ ] File saved

---

## 3. Swap the GoogleService-Info file

The iOS project keeps three `GoogleService-Info` plists side by side — the active one Xcode reads, plus a dev and a prod copy kept in the repo for reference:

```
ios
├─ AppDelegate.mm
├─ GoogleService-Info.plist          ← active file Xcode reads
├─ GoogleServiceDevFR-Info.plist     dev copy
├─ GoogleServiceProd-Info.plist      prod copy
├─ Podfile
├─ Podfile.lock
├─ foodrelatedDevFR-Info.plist
├─ foodrelatedDevFR.entitlements
├─ foodrelatedProd-Info.plist
└─ foodrelatedProd.entitlements
```

![ios folder listing showing the GoogleService-Info plist files](assets/image3.jpg)

Replace `GoogleService-Info.plist` with a copy of `GoogleServiceProd-Info.plist` before releasing, so the build ships pointed at production Firebase.

---

## 4. Bump the Android version (`build.gradle`)

Open the `android` folder inside `app` and find `build.gradle`. The `production` flavor under `productFlavors` is what ships to the Play Store.

```groovy
productFlavors {
  staging {
    applicationId "com.foodrelated.devFR"
    versionCode 1
    versionName "1.0"
    applicationIdSuffix ".stage"
  }
  production {
    applicationId "com.foodrelated.release"
    // increment both together
    versionCode 1066
    versionName "1.0.66"
    applicationIdSuffix ".prod"
  }
}
```

```diff
- versionCode 1065 · versionName "1.0.65"
+ versionCode 1066 · versionName "1.0.66"
```

> ⚠️ **Always increment, never reuse.** If a build gets rejected in the Play Console, the next upload still needs a new, higher `versionCode` — the Play Store will not accept a repeat.

Where `build.gradle` sits, and what a built `android/app/build` output folder looks like once you've run a release build:

```
android/app
├─ build
│  ├─ crashlytics
│  │  ├─ productionRelease/mappingFileId.txt
│  │  └─ stagingRelease/mappingFileId.txt
│  ├─ generated
│  │  └─ ap_generated_sources
│  │     ├─ productionRelease/out
│  │     └─ stagingRelease/out
│  ├─ assets
│  │  ├─ createBundleProductionReleaseJsAndAssets/index.android.bundle
│  │  └─ createBundleStagingReleaseJsAndAssets/index.android.bundle
│  └─ autolinking/src/main/java/com/facebook/react/PackageList.java
└─ build.gradle
```

![File tree leading to build.gradle](assets/image16.jpg)

---

## 5. Point at the release keystore

In `gradle.properties`, comment out the debug signing lines and uncomment the release keystore lines so the production build is signed correctly.

```properties
44: MYAPP_UPLOAD_STORE_FILE=fr-aveto-key.keystore
45: MYAPP_UPLOAD_KEY_ALIAS=fr-aveto-key-alias
46: # for prod uncomment below
47: # MYAPP_UPLOAD_STORE_FILE=my-upload-key.keystore
48: # MYAPP_UPLOAD_KEY_ALIAS=my-key-alias
49: MYAPP_UPLOAD_STORE_PASSWORD=••••••••
50: MYAPP_UPLOAD_KEY_PASSWORD=••••••••
```

![gradle.properties showing the debug and release keystore lines](assets/image13.jpg)

**Checklist**
- [ ] Comment out line 44 and line 45 (debug keystore)
- [ ] Uncomment line 47 and line 48 (release keystore)

---

## 6. Xcode: build, archive, distribute

1. Open the project in Xcode — select the **`foodrelated.xcworkspace`** file, not `.xcodeproj`.
2. In the scheme selector, pick the **`foodrelatedProd`** scheme (the project also defines `foodrelated`, `foodrelatedTests`, and `foodrelatedDevFR` — make sure prod is selected).
3. Set the run destination to **Any iOS Device (arm64)**.
4. In the target's **General** tab, confirm/update **Version** and **Build** (Bundle Identifier is `com.foodrelated`, Minimum Deployment `iOS 13.4`), and press Enter after editing.
5. **Product → Clean Build Folder.**
6. **Product → Build**, and wait for it to complete (status bar shows *Build Succeeded*).
7. **Product → Archive**, and wait for it to complete.
8. From the **Organizer**, select the new archive and click **Distribute App** to send the build to App Store Connect.
9. Once it finishes uploading, click **Validate App** — the archive shows *Uploaded to Apple* with its version, build number, bundle identifier, team, and architecture (`arm64`).
10. In App Store Connect, the build appears under **TestFlight** first; once ready, it's attached to the release under **Distribution** (**+** icon → new release version).

```
git branch: release/prodRelease-1.4.6
Scheme:     foodrelatedProd
Bundle ID:  com.foodrelated
Team:       RJL Texas International...
Min iOS:    13.4
Arch:       arm64
```

| Step | Screenshot |
|---|---|
| Open the `.xcworkspace`, not `.xcodeproj` | ![Xcode workspace selection](assets/image21.jpg) |
| Select the `foodrelatedProd` scheme | ![Scheme selector: foodrelated, foodrelatedTests, foodrelatedDevFR, foodrelatedProd](assets/image8.jpg) |
| Destination: Any iOS Device (arm64) | ![Run destination list including simulators and physical devices](assets/image11.jpg) |
| Confirm version in the General/Identity panel | ![General tab showing Version 1.0.65, Build 1, Bundle Identifier com.foodrelated](assets/image5.jpg) |
| Product → Clean Build Folder | ![Product menu with Clean Build Folder highlighted](assets/image4.jpg) |
| Build — wait for "Build Succeeded" | ![Xcode build in progress](assets/image17.jpg) |
| Archive | ![Xcode archive step](assets/image15.jpg) |
| Organizer → Distribute App | ![Archives panel: foodrelatedProd, Uploaded to Apple](assets/image20.jpg) |
| Validate → build appears under TestFlight | ![App Store Connect Builds screen listing versions](assets/image2.jpg) |
| Distribution → + to attach the release | ![App Store Connect version 1.4.6 Ready for Distribution](assets/image19.jpg) |

📹 Video walkthrough: [AddIOS.mov](https://drive.google.com/file/d/1EKkyBqTLWy709z2FohJg5c3F-ewyOLf9/view?usp=drive_link)

---

## 7. Android: build & publish to Play Console

1. In Android Studio, select the production run configuration — bundle ID `com.foodrelated.release.prod`.
2. In the code editor's terminal, on the release branch (e.g. `release/prodRelease-1.4.6`), run:

   ```bash
   yarn android-prod-release
   ```

3. Once the build finishes, open `android/app/build/outputs` and locate the generated **`.aab`** bundle (a `productionRelease` document, roughly 30 MB).
4. In the [Play Console](https://play.google.com/console), open the app and go to **Test and release → Production**, then **New release** (or check **Release dashboard** for the current live version and rollout stats).
5. Under **Upload app bundles**, drag in the `.aab`, or **Add from library**.
6. Fill in the **Release name** (convention: `<version>-Release`, e.g. `1.4.6-Release`) and **Release notes** for each locale (e.g. `<en-US>…</en-US>`).
7. Click **Next**, review the rollout, and publish.

| Step | Screenshot |
|---|---|
| Select the production run config | ![Android Studio target selection](assets/image6.jpg) |
| Confirm the production flavor | ![Android production flavor selection continued](assets/image18.jpg) |
| `yarn android-prod-release` in the terminal, on the release branch | *(same as above — terminal pane)* |
| Locate the build output / project structure | ![Android build folder tree with keystores and flavors](assets/image7.jpg) |
| Generated `.aab` bundle info | ![Finder info panel: productionRelease document, 31.2 MB](assets/image9.jpg) |
| Play Console → Production → release dashboard | ![Play Console release dashboard showing 1.4.6-Release live](assets/image10.jpg) |
| Play Console app list (all package variants) | ![Play Console showing FR Wholesale, Food Related, staging and dev variants](assets/image1.jpg) |
| New release: name + notes | ![Create production release form with release name and notes fields](assets/image12.jpg) |

📹 Video walkthroughs:
- [AndroidIOSRelease.mov](https://drive.google.com/file/d/1D_Vc_cYGKGmKzckQHjS5E_gHhlok4FHd/view?usp=drive_link)
- [androidplayconsolevideo.mov](https://drive.google.com/file/d/1HW_VolDm6YIRmCBEHOjorjE3LTZbURhc/view?usp=drive_link)

---

## 8. Reference: identifiers & naming conventions

Pulled directly from the Play Console / App Store Connect / Xcode screenshots — useful for double-checking you're pointed at the right app variant before releasing.

### Play Console apps (same org, different build variants)

| App name | Package | Status seen |
|---|---|---|
| FR Wholesale | `com.foodrelated.foodrelatedapp` | Production |
| **Food Related** (prod) | `com.foodrelated.release.prod` | Production ← the one this runbook releases |
| Food Related (draft) | `com.foodrelated.stage` | Unpublished |
| FoodRelated (internal) | `com.foodrelated.staging.stage` | Internal testing |
| S-FoodRelated (dev) | `com.foodrelated.devFR.stage` | Unpublished |

### iOS identifiers

| Item | Value |
|---|---|
| Bundle Identifier | `com.foodrelated` |
| Minimum Deployment | iOS 13.4 |
| Schemes | `foodrelated`, `foodrelatedTests`, `foodrelatedDevFR`, `foodrelatedProd` |
| Release branch pattern | `release/prodRelease-<version>` |
| Team | RJL Texas International... |

### Versioning note

The two platforms track versions independently:
- **JS-level `version`** in `utils.js` (e.g. `1.4.6`) is the app's marketing/release version shown in the Play Console and App Store Connect ("iOS App Version 1.4.6").
- **Android `versionCode` / `versionName`** in `build.gradle` increments separately (e.g. `1066` / `1.0.66`) and must always go up, even across a rejected build.
- **iOS Build number** (in Xcode's General tab, alongside Version) is a separate incrementing counter per upload — App Store Connect shows both the marketing version and the archive's build number side by side.

---

*Converted from the original release notes document. Screenshots referenced above live in [`assets/`](./assets).*
