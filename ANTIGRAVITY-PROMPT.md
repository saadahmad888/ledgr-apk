# Prompt for Antigravity / Gemini

Copy everything inside the box below and paste it as your first message in Antigravity,
with the `ledgr-apk` repo open.

---

```
You are working in the root of my GitHub repo `ledgr-apk` (branch: main). It is a static
site published with GitHub Pages at https://saadahmad888.github.io/ledgr-apk/

GOAL
The root URL must open a public landing page where a visitor chooses between downloading
the Android APK and opening the app in the browser. The app itself must live at /app.html.
Right now the app may still be sitting at /index.html — if so, move it.

REQUIRED FINAL FILE LAYOUT (all at repo root, no subfolders)
  index.html                 landing / download page
  app.html                   the actual expense-tracker app
  manifest.webmanifest
  sw.js
  icon-192.png
  icon-512.png
  icon-maskable-512.png
  apple-touch-icon.png
  .nojekyll                  empty file, must exist
  README.md

TASKS
1. If a file named index.html contains the expense tracker app (look for `const DB=` and
   `openTxSheet`), rename it to app.html. Do not edit its contents or reformat it.
2. Make sure index.html is the landing page (look for the text "Download the APK"). If it
   is missing, tell me and stop — do not invent one.
3. In index.html, the secondary button with id="pwa" must link to "app.html", not
   "index.html".
4. In manifest.webmanifest set:
       "start_url": "./app.html"
       "scope": "./"
   and both entries under "shortcuts" must use "./app.html#add" and "./app.html#note".
   Keep the file as valid JSON.
5. In sw.js:
       - the ASSETS array must include both './index.html' and './app.html'
       - the offline fallback at the end must be caches.match('./app.html')
       - bump the CACHE constant by one version (for example 'ledgr-v5' -> 'ledgr-v6')
6. Create .nojekyll at the root if it does not exist. It must be an empty file. This is
   required or GitHub Pages hides dot-folders.
7. Add a .gitignore containing these lines, so I can never leak my signing key:
       *.keystore
       *.jks
       *.apk
       *.aab
       keystore.txt
       key-info.txt

RULES
- Do not change any application logic, CSS, or wording inside app.html.
- Do not add build tools, frameworks, npm packages, bundlers or a package.json.
  This is deliberately a dependency-free static site.
- Keep every path relative (./file), never absolute (/file), because the site is served
  from a /ledgr-apk/ subfolder.
- Do not delete or rewrite README.md.

VERIFY BEFORE COMMITTING
- manifest.webmanifest parses as JSON and start_url is ./app.html
- grep app.html in sw.js returns a hit inside ASSETS
- index.html contains href="app.html"
- app.html exists and no index.html contains `openTxSheet`
- .nojekyll exists and is 0 bytes
Print the output of `git status --short` and a one-line summary of each file you changed.

THEN COMMIT AND PUSH
  git add -A
  git commit -m "Landing page at root, app moved to app.html, PWA wiring updated"
  git push origin main

Finally, tell me the two URLs I should test in my phone browser and what I should see at
each one.
```

---

## If you would rather skip the agent

The files I gave you are already in this exact state, so you can just do it yourself:

```bash
cd path/to/ledgr-apk

# copy the files from the zip over the repo, then:
git add -A
git commit -m "Landing page at root, app moved to app.html, PWA wiring updated"
git push origin main
```

## What to test afterwards

| URL | What you should see |
|---|---|
| `https://saadahmad888.github.io/ledgr-apk/` | The landing page, with a Download the APK button |
| `https://saadahmad888.github.io/ledgr-apk/app.html` | The app itself, greeting and green Add button |

Then feed **the second URL** to pwabuilder.com. The landing page has no manifest, so
PWABuilder would reject the first one.

---

© Saad Ahmad · [isaadahmad.com](https://isaadahmad.com)
