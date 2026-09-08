# Ledgr — offline expenses, notes & goals

**Built by Saad Ahmad — [isaadahmad.com](https://isaadahmad.com)**

An offline-first personal finance app: daily expenses by category, receipt photos,
voice notes and checklists, savings goals, stats, and PDF / Excel / CSV export and import.
No account, no server, no tracking. Everything is stored on the phone.

Works in **125 countries** — currency, number format, week start and payment methods all
follow wherever you are.

---

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app. UI, logic, database, PDF and Excel engines. No libraries, no CDN. |
| `manifest.webmanifest` | Makes it installable with a name, icon and standalone window. |
| `sw.js` | Service worker. Caches the app so it opens with the network completely off. |
| `icon-192.png` `icon-512.png` `icon-maskable-512.png` `apple-touch-icon.png` | App icons. |

Keep all files in the same folder. Paths are relative, so a subfolder works fine.

---

## 1. Host it on GitHub Pages (free, and it gives you the HTTPS link you need)

**Through the website, no command line:**

1. Sign in at github.com → **+** (top right) → **New repository**.
2. Name it `ledgr`, set it to **Public**, click **Create repository**.
3. On the empty repo page click **uploading an existing file**.
4. Drag in all eight files (`index.html`, `manifest.webmanifest`, `sw.js`, the four icons,
   `README.md`). They must sit at the top level, not inside a folder.
5. Click **Commit changes**.
6. Go to **Settings → Pages**. Under *Build and deployment* set
   **Source: Deploy from a branch**, **Branch: `main`**, **Folder: `/ (root)`** → **Save**.
7. Wait a minute or two, then reload that page. Your link appears at the top:
   `https://YOUR-USERNAME.github.io/ledgr/`

**Or with git:**

```bash
git init
git add .
git commit -m "Ledgr v1"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/ledgr.git
git push -u origin main
```

Then do step 6 above.

### Putting it on your own domain

In **Settings → Pages → Custom domain**, enter `ledgr.isaadahmad.com` and save.
Then at your DNS provider add a **CNAME** record:

| Type | Name | Value |
|---|---|---|
| CNAME | `ledgr` | `YOUR-USERNAME.github.io` |

Come back once DNS propagates and tick **Enforce HTTPS**. GitHub commits a `CNAME`
file to the repo automatically — leave it there.

### Updating later

Upload the changed `index.html` over the old one, and bump `CACHE` in `sw.js`
(`ledgr-v2` → `ledgr-v3`). Without that bump, phones keep serving the cached copy.

---

## 2. Install it as an app

Open your GitHub Pages link in Chrome on the phone → menu (⋮) → **Install app** /
**Add to Home screen**. Turn off mobile data and open it from the home screen: full
screen, no browser bars, everything works.

It has to be served over `https://`. Opening the file straight from your Downloads
folder shows the app, but browsers block storage on `file://` addresses so nothing
saves — the app warns you if that happens.

## 3. Turn it into a signed APK, no coding (PWABuilder)

1. Go to **pwabuilder.com**, paste your GitHub Pages URL.
2. **Package for stores → Android**.
3. Set the package ID to something like `com.isaadahmad.ledgr`.
4. Download the zip → inside is `app-release-signed.apk`.
5. Copy it to the phone, tap it, allow "install from unknown sources".

## 4. Fully offline APK with no hosting (Android Studio, ~10 min)

This bundles the app inside the APK, so nothing is ever fetched from the internet.

1. Android Studio → **New Project → Empty Views Activity**, Kotlin.
2. Put `index.html`, `sw.js`, `manifest.webmanifest` and the icons in
   `app/src/main/assets/ledgr/`.
3. `app/build.gradle` dependencies: `implementation "androidx.webkit:webkit:1.11.0"`
4. `AndroidManifest.xml` — inside `<manifest>`:
   ```xml
   <uses-permission android:name="android.permission.RECORD_AUDIO"/>
   <uses-permission android:name="android.permission.INTERNET"/>
   ```
   and inside `<application>`:
   ```xml
   <provider
       android:name="androidx.core.content.FileProvider"
       android:authorities="${applicationId}.fileprovider"
       android:exported="false"
       android:grantUriPermissions="true">
       <meta-data android:name="android.support.FILE_PROVIDER_PATHS"
                  android:resource="@xml/file_paths"/>
   </provider>
   ```
5. `app/src/main/res/xml/file_paths.xml`:
   ```xml
   <paths><cache-path name="shots" path="."/></paths>
   ```
6. `activity_main.xml` — one full-screen WebView with `android:id="@+id/web"`.
7. `MainActivity.kt`:

```kotlin
import android.Manifest
import android.content.Intent
import android.net.Uri
import android.os.Bundle
import android.provider.MediaStore
import android.webkit.*
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.app.ActivityCompat
import androidx.core.content.FileProvider
import androidx.webkit.WebViewAssetLoader
import java.io.File

class MainActivity : AppCompatActivity() {

    private lateinit var web: WebView
    private var fileCallback: ValueCallback<Array<Uri>>? = null
    private var cameraUri: Uri? = null

    private val picker = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { r ->
        val fromCamera = r.data == null && r.resultCode == RESULT_OK
        val uris = when {
            r.resultCode != RESULT_OK -> null
            fromCamera && cameraUri != null -> arrayOf(cameraUri!!)
            else -> WebChromeClient.FileChooserParams.parseResult(r.resultCode, r.data)
        }
        fileCallback?.onReceiveValue(uris)
        fileCallback = null
        cameraUri = null
    }

    override fun onCreate(saved: Bundle?) {
        super.onCreate(saved)
        setContentView(R.layout.activity_main)
        web = findViewById(R.id.web)

        ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.RECORD_AUDIO), 1)

        // Serving from https://appassets.androidplatform.net gives the page a secure
        // origin, which is what makes IndexedDB and the service worker work.
        val loader = WebViewAssetLoader.Builder()
            .addPathHandler("/assets/", WebViewAssetLoader.AssetsPathHandler(this))
            .build()

        web.settings.apply {
            javaScriptEnabled = true
            domStorageEnabled = true
            databaseEnabled = true
            allowFileAccess = false
            mediaPlaybackRequiresUserGesture = false
        }

        web.webViewClient = object : WebViewClient() {
            override fun shouldInterceptRequest(v: WebView, req: WebResourceRequest) =
                loader.shouldInterceptRequest(req.url)
        }

        web.webChromeClient = object : WebChromeClient() {
            override fun onPermissionRequest(req: PermissionRequest) = req.grant(req.resources)

            override fun onShowFileChooser(
                v: WebView, cb: ValueCallback<Array<Uri>>, params: FileChooserParams
            ): Boolean {
                fileCallback?.onReceiveValue(null)
                fileCallback = cb
                val photo = File.createTempFile("receipt", ".jpg", cacheDir)
                cameraUri = FileProvider.getUriForFile(
                    this@MainActivity, "$packageName.fileprovider", photo
                )
                val camera = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
                    .putExtra(MediaStore.EXTRA_OUTPUT, cameraUri)
                val chooser = Intent.createChooser(params.createIntent(), "Receipt photo")
                    .putExtra(Intent.EXTRA_INITIAL_INTENTS, arrayOf(camera))
                picker.launch(chooser)
                return true
            }
        }

        web.loadUrl("https://appassets.androidplatform.net/assets/ledgr/index.html")
    }

    override fun onBackPressed() {
        if (web.canGoBack()) web.goBack() else super.onBackPressed()
    }
}
```

8. **Build → Generate Signed Bundle / APK → APK**, create a keystore, choose `release`.
   The APK lands in `app/release/`.

---

## Region support

Choose your country once, in the intro or under **More → Region & money**:

- **Currency** — 90-plus currencies, formatted the way your country writes numbers.
  Zero-decimal currencies such as the yen, won and dong are handled correctly.
  Currency can be set independently of country, and the symbol can be overridden by hand.
- **Payment methods** — the ones people actually use where you live. UPI, PhonePe and
  Paytm in India. Easypaisa, JazzCash and Raast in Pakistan. Pix and Boleto in Brazil.
  M-Pesa in Kenya and Tanzania. Zelle, Venmo and Cash App in the US. iDEAL in the
  Netherlands, BLIK in Poland, Swish in Sweden, TWINT in Switzerland, PayNow in Singapore,
  GCash in the Philippines, QRIS and GoPay in Indonesia, Mada and STC Pay in Saudi Arabia,
  Alipay and WeChat Pay in China, and so on. The list is fully editable — add, remove,
  reorder, or reset to your country's defaults.
- **Week start and date format** follow your locale, so the calendar starts on the right day.
- **Voice language** is set from your country, changeable from 70-plus options.

## Where your data lives

Everything sits in the app's own IndexedDB on the phone: transactions, categories, receipt
photos (resized to roughly 100 KB each), notes, goals and settings. There is no sync and no
cloud backup — that is the point, and also the risk. Use **More → Full backup file** now and
then and keep the `.json` somewhere safe. That one file restores everything, receipts
included, on any phone.

## One limitation worth knowing

Voice-to-text uses the browser's speech API, which on Android sends audio to Google's
service, so **that one feature needs a connection**. Everything else works with data off.
Offline, use the microphone key on your phone's own keyboard instead — most Android
keyboards do it on-device.

---

Designed and developed by **Saad Ahmad** · [isaadahmad.com](https://isaadahmad.com)
