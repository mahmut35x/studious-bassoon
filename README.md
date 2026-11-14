# Salon Launcher: bu tek dosya projeyi oluşturur, derler ve APK'yı Releases bölümüne koyar.
name: Salon Launcher APK
on:
  push:
    branches: [main, master]
  workflow_dispatch:
permissions:
  contents: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'
      - name: Android lisanslarını kabul et
        run: |
          yes | sdkmanager --licenses > /dev/null || true
      - name: Proje dosyalarını oluştur
        run: |
          set -e
          mkdir -p salon salon/app salon/app/src/main salon/app/src/main/assets salon/app/src/main/java/com/salon/launcher salon/app/src/main/res/drawable
          cat > salon/settings.gradle.kts <<'SALON_EOF'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "SalonLauncher"
          include(":app")
          SALON_EOF
          cat > salon/build.gradle.kts <<'SALON_EOF'
          plugins {
              id("com.android.application") version "8.5.2" apply false
          }
          SALON_EOF
          cat > salon/gradle.properties <<'SALON_EOF'
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=false
          android.nonTransitiveRClass=true
          SALON_EOF
          cat > salon/app/build.gradle.kts <<'SALON_EOF'
          plugins {
              id("com.android.application")
          }

          android {
              namespace = "com.salon.launcher"
              compileSdk = 34

              defaultConfig {
                  applicationId = "com.salon.launcher"
                  minSdk = 21
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }

              buildTypes {
                  release {
                      isMinifyEnabled = false
                  }
              }

              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_1_8
                  targetCompatibility = JavaVersion.VERSION_1_8
              }
          }
          SALON_EOF
          cat > salon/app/src/main/AndroidManifest.xml <<'SALON_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-feature
                  android:name="android.software.leanback"
                  android:required="false" />
              <uses-feature
                  android:name="android.hardware.touchscreen"
                  android:required="false" />

              <uses-permission android:name="android.permission.QUERY_ALL_PACKAGES" />

              <application
                  android:allowBackup="false"
                  android:label="Salon"
                  android:icon="@drawable/ic_launcher"
                  android:banner="@drawable/banner"
                  android:hardwareAccelerated="true"
                  android:theme="@android:style/Theme.NoTitleBar.Fullscreen">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:launchMode="singleTask"
                      android:screenOrientation="landscape"
                      android:configChanges="orientation|screenSize|keyboard|keyboardHidden|navigation|uiMode">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.HOME" />
                          <category android:name="android.intent.category.DEFAULT" />
                          <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
                      </intent-filter>
                  </activity>

              </application>
          </manifest>
          SALON_EOF
          cat > salon/app/src/main/java/com/salon/launcher/MainActivity.java <<'SALON_EOF'
          package com.salon.launcher;

          import android.app.Activity;
          import android.content.Intent;
          import android.content.SharedPreferences;
          import android.content.pm.PackageManager;
          import android.content.pm.ResolveInfo;
          import android.graphics.Bitmap;
          import android.graphics.Canvas;
          import android.graphics.Color;
          import android.graphics.drawable.Drawable;
          import android.net.Uri;
          import android.os.Bundle;
          import android.view.View;
          import android.webkit.JavascriptInterface;
          import android.webkit.WebResourceRequest;
          import android.webkit.WebResourceResponse;
          import android.webkit.WebSettings;
          import android.webkit.WebView;
          import android.webkit.WebViewClient;

          import org.json.JSONArray;
          import org.json.JSONObject;

          import java.io.ByteArrayInputStream;
          import java.io.ByteArrayOutputStream;
          import java.io.InputStream;
          import java.text.Collator;
          import java.util.ArrayList;
          import java.util.Collections;
          import java.util.Comparator;
          import java.util.HashMap;
          import java.util.HashSet;
          import java.util.List;
          import java.util.Locale;
          import java.util.Map;
          import java.util.Set;

          public class MainActivity extends Activity {

              private WebView web;
              private SharedPreferences prefs;

              @Override
              protected void onCreate(Bundle savedInstanceState) {
                  super.onCreate(savedInstanceState);
                  prefs = getSharedPreferences("salon", MODE_PRIVATE);

                  web = new WebView(this);
                  web.setBackgroundColor(Color.parseColor("#05070C"));
                  web.setOverScrollMode(View.OVER_SCROLL_NEVER);
                  web.setVerticalScrollBarEnabled(false);
                  web.setHorizontalScrollBarEnabled(false);
                  setContentView(web);

                  WebSettings settings = web.getSettings();
                  settings.setJavaScriptEnabled(true);
                  settings.setDomStorageEnabled(true);
                  settings.setAllowFileAccess(false);
                  settings.setAllowContentAccess(false);

                  web.addJavascriptInterface(new Bridge(), "Android");
                  web.setWebViewClient(new WebViewClient() {
                      @Override
                      public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
                          return handleRequest(request.getUrl());
                      }

                      @Override
                      public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
                          return true;
                      }

                      @SuppressWarnings("deprecation")
                      @Override
                      public boolean shouldOverrideUrlLoading(WebView view, String url) {
                          return true;
                      }
                  });

                  web.loadDataWithBaseURL("https://salon.local/", readAsset("index.html"), "text/html", "utf-8", null);
                  web.requestFocus();
              }

              @Override
              protected void onResume() {
                  super.onResume();
                  if (web != null) {
                      web.evaluateJavascript("window.salonRefresh&&window.salonRefresh()", null);
                  }
              }

              @Override
              protected void onNewIntent(Intent intent) {
                  super.onNewIntent(intent);
                  if (web != null) {
                      web.evaluateJavascript("window.salonHome&&window.salonHome()", null);
                  }
              }

              @SuppressWarnings("deprecation")
              @Override
              public void onBackPressed() {
                  // Ana ekranda geri tuşu hiçbir şey yapmaz.
              }

              @Override
              protected void onDestroy() {
                  if (web != null) {
                      web.destroy();
                      web = null;
                  }
                  super.onDestroy();
              }

              private String readAsset(String name) {
                  try {
                      InputStream in = getAssets().open(name);
                      ByteArrayOutputStream out = new ByteArrayOutputStream();
                      byte[] buf = new byte[8192];
                      int n;
                      while ((n = in.read(buf)) > 0) {
                          out.write(buf, 0, n);
                      }
                      in.close();
                      return out.toString("UTF-8");
                  } catch (Exception e) {
                      return "<h1>index.html okunamadi</h1>";
                  }
              }

              private WebResourceResponse handleRequest(Uri uri) {
                  if ("salon.local".equals(uri.getHost())) {
                      String path = uri.getPath();
                      if (path != null && path.startsWith("/icon/")) {
                          WebResourceResponse r = icon(path.substring(6));
                          if (r != null) {
                              return r;
                          }
                      }
                  }
                  return new WebResourceResponse("text/plain", "utf-8", 404, "Not Found",
                          new HashMap<String, String>(), new ByteArrayInputStream(new byte[0]));
              }

              private WebResourceResponse icon(String pkg) {
                  try {
                      Drawable d = getPackageManager().getApplicationIcon(pkg);
                      int size = 144;
                      Bitmap bm = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888);
                      Canvas c = new Canvas(bm);
                      d.setBounds(0, 0, size, size);
                      d.draw(c);
                      ByteArrayOutputStream out = new ByteArrayOutputStream();
                      bm.compress(Bitmap.CompressFormat.PNG, 100, out);
                      bm.recycle();
                      Map<String, String> headers = new HashMap<String, String>();
                      headers.put("Cache-Control", "max-age=3600");
                      return new WebResourceResponse("image/png", null, 200, "OK", headers,
                              new ByteArrayInputStream(out.toByteArray()));
                  } catch (Exception e) {
                      return null;
                  }
              }

              private Set<String> hiddenSet() {
                  return new HashSet<String>(prefs.getStringSet("hidden", new HashSet<String>()));
              }

              public class Bridge {

                  @JavascriptInterface
                  public String apps() {
                      JSONArray arr = new JSONArray();
                      try {
                          PackageManager pm = getPackageManager();
                          Set<String> hidden = hiddenSet();
                          Map<String, String> found = new HashMap<String, String>();
                          String[] cats = {Intent.CATEGORY_LEANBACK_LAUNCHER, Intent.CATEGORY_LAUNCHER};
                          for (String cat : cats) {
                              Intent query = new Intent(Intent.ACTION_MAIN);
                              query.addCategory(cat);
                              List<ResolveInfo> list = pm.queryIntentActivities(query, 0);
                              for (ResolveInfo ri : list) {
                                  String pkg = ri.activityInfo.packageName;
                                  if (pkg.equals(getPackageName()) || hidden.contains(pkg) || found.containsKey(pkg)) {
                                      continue;
                                  }
                                  found.put(pkg, String.valueOf(ri.loadLabel(pm)));
                              }
                          }
                          List<Map.Entry<String, String>> sorted =
                                  new ArrayList<Map.Entry<String, String>>(found.entrySet());
                          final Collator collator = Collator.getInstance(new Locale("tr", "TR"));
                          Collections.sort(sorted, new Comparator<Map.Entry<String, String>>() {
                              @Override
                              public int compare(Map.Entry<String, String> a, Map.Entry<String, String> b) {
                                  return collator.compare(a.getValue(), b.getValue());
                              }
                          });
                          for (Map.Entry<String, String> e : sorted) {
                              JSONObject o = new JSONObject();
                              o.put("p", e.getKey());
                              o.put("l", e.getValue());
                              arr.put(o);
                          }
                      } catch (Exception ex) {
                          // Hata olursa bos liste doner.
                      }
                      return arr.toString();
                  }

                  @JavascriptInterface
                  public int hiddenCount() {
                      return hiddenSet().size();
                  }

                  @JavascriptInterface
                  public void launch(String pkg) {
                      try {
                          PackageManager pm = getPackageManager();
                          Intent i = pm.getLeanbackLaunchIntentForPackage(pkg);
                          if (i == null) {
                              i = pm.getLaunchIntentForPackage(pkg);
                          }
                          if (i != null) {
                              i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                              startActivity(i);
                          }
                      } catch (Exception e) {
                          // Acilamayan uygulama sessizce atlanir.
                      }
                  }

                  @JavascriptInterface
                  public void hide(String pkg) {
                      Set<String> s = hiddenSet();
                      s.add(pkg);
                      prefs.edit().putStringSet("hidden", s).apply();
                  }

                  @JavascriptInterface
                  public void unhideAll() {
                      prefs.edit().remove("hidden").apply();
                  }

                  @JavascriptInterface
                  public String getPref(String key) {
                      return prefs.getString("p_" + key, "");
                  }

                  @JavascriptInterface
                  public void setPref(String key, String value) {
                      prefs.edit().putString("p_" + key, value).apply();
                  }
              }
          }
          SALON_EOF
          cat > salon/app/src/main/res/drawable/ic_launcher.xml <<'SALON_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android"
              android:shape="rectangle">
              <gradient
                  android:angle="315"
                  android:startColor="#7AA8FF"
                  android:endColor="#7B4DFF" />
              <corners android:radius="48dp" />
              <size
                  android:width="192dp"
                  android:height="192dp" />
          </shape>
          SALON_EOF
          cat > salon/app/src/main/res/drawable/banner.xml <<'SALON_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android"
              android:shape="rectangle">
              <gradient
                  android:angle="315"
                  android:startColor="#3D6BFF"
                  android:endColor="#8A4DFF" />
              <size
                  android:width="320dp"
                  android:height="180dp" />
          </shape>
          SALON_EOF
          cat > salon/app/src/main/assets/index.html <<'SALON_EOF'
          <!doctype html>
          <html lang="tr">
          <head>
          <meta charset="utf-8">
          <meta name="viewport" content="width=device-width,initial-scale=1">
          <title>Salon</title>
          <style>
          :root{--bg:#05070c;--fg:#f6f8ff;--muted:rgba(236,241,255,.72);--hue:215;--tint:.12;color-scheme:dark}
          *{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
          html,body{height:100%;margin:0;overflow:hidden}
          body{background:var(--bg);color:var(--fg);font-family:Roboto,"Segoe UI",system-ui,sans-serif;font-size:16px;line-height:1.4;-webkit-user-select:none;user-select:none}
          [hidden]{display:none!important}

          /* Duvar kagidi: odaktaki uygulamanin rengine gore degisir */
          .wall{position:fixed;top:0;left:0;right:0;bottom:0;z-index:-1;background:
            radial-gradient(46% 46% at 16% 18%,hsla(var(--hue),92%,56%,.85),transparent 72%),
            radial-gradient(40% 42% at 86% 12%,hsla(calc(var(--hue) + 70),90%,62%,.7),transparent 72%),
            radial-gradient(52% 52% at 72% 88%,hsla(calc(var(--hue) - 60),86%,52%,.75),transparent 72%),
            var(--bg)}

          .stage{height:100%;display:flex;flex-direction:column;padding:24px 48px 0}
          header.top{flex:0 0 auto;display:flex;align-items:center;justify-content:space-between}
          .left{display:flex;align-items:center}
          .left .glass{margin-right:12px}

          /* Cam malzeme. Hafif modda bulaniklik yok, sadece yari saydam dolgu ve parlak kenar */
          .glass{position:relative;z-index:0;background:linear-gradient(135deg,rgba(255,255,255,.2),rgba(255,255,255,.07));
            box-shadow:inset 0 1px 0 rgba(255,255,255,.55),inset 0 -1px 0 rgba(0,0,0,.3),inset 0 0 0 1px rgba(255,255,255,.12),0 14px 26px -14px rgba(0,0,0,.6)}
          .glass::before{content:"";position:absolute;left:0;top:0;right:0;bottom:0;border-radius:inherit;background:rgba(14,18,30,.82);opacity:var(--tint);z-index:-1}
          .blur .glass{-webkit-backdrop-filter:blur(18px) saturate(1.6);backdrop-filter:blur(18px) saturate(1.6)}

          .pill{display:flex;align-items:center;padding:10px 20px;border-radius:999px;font-weight:700}
          .brand{font-size:13px;letter-spacing:.16em}
          .btn{outline:none;transition:transform .15s ease-out}
          .btn .k{font-size:13px;font-weight:600;opacity:.7;margin-right:10px}
          .btn .v{font-size:14px}
          .btn:focus{transform:scale(1.06);box-shadow:inset 0 1px 0 rgba(255,255,255,.7),0 0 0 3px #fff,0 16px 28px -12px rgba(0,0,0,.7)}
          #time{font-size:22px;line-height:1;margin-right:12px;font-variant-numeric:tabular-nums}
          #date{font-size:14px;font-weight:500;color:var(--muted)}

          .hero{flex:0 0 auto;padding:22px 0 6px;min-height:118px;text-shadow:0 2px 24px rgba(0,0,0,.4)}
          .kicker{margin:0 0 6px;font-size:13px;letter-spacing:.14em;text-transform:uppercase;font-weight:700;color:rgba(255,255,255,.82)}
          .hero h1{margin:0;font-size:44px;font-size:clamp(30px,4.4vw,60px);font-weight:800;letter-spacing:-.03em;line-height:1.05;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
          .hero p{margin:8px 0 0;color:var(--muted);font-size:16px;font-size:clamp(14px,1.4vw,19px)}

          .scroll{flex:1 1 auto;overflow-y:auto;overflow-x:hidden;padding:26px 30px 40px;margin:0 -30px;scrollbar-width:none}
          .scroll::-webkit-scrollbar{display:none}
          .grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(112px,1fr));grid-template-columns:repeat(auto-fill,minmax(clamp(100px,9.5vw,150px),1fr));grid-gap:30px 20px;gap:30px 20px}
          .empty{color:var(--muted);padding:20px 4px}

          .app{position:relative;outline:none;text-align:center;transition:transform .16s ease-out}
          .sq{position:relative;display:block;border-radius:24%;overflow:hidden;background:linear-gradient(155deg,rgba(255,255,255,.22),rgba(255,255,255,.06));
            box-shadow:inset 0 1.5px 0 rgba(255,255,255,.7),inset 0 0 0 1px rgba(255,255,255,.16),0 12px 20px -10px rgba(0,0,0,.7)}
          .sq::before{content:"";display:block;padding-top:100%}
          .sq img{position:absolute;left:0;top:0;width:100%;height:100%;object-fit:cover}
          .sq::after{content:"";position:absolute;left:0;top:0;right:0;bottom:0;pointer-events:none;background:radial-gradient(90% 70% at 25% 6%,rgba(255,255,255,.38),transparent 56%)}
          .sq.ph{display:flex;align-items:center;justify-content:center;font-size:40px;font-weight:800;background:linear-gradient(155deg,hsla(var(--h),85%,62%,1),hsla(calc(var(--h) + 25),80%,38%,1))}
          .lbl{display:block;margin-top:10px;font-size:15px;font-size:clamp(13px,1.15vw,17px);font-weight:600;color:var(--muted);white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
          .app:focus{transform:scale(1.12) translateY(-4px);z-index:2}
          .app:focus .sq{box-shadow:inset 0 2px 0 rgba(255,255,255,.95),0 0 0 3px rgba(255,255,255,.95),0 22px 32px -12px rgba(0,0,0,.85)}
          .app:focus .lbl{color:#fff}

          .hint{flex:0 0 auto;padding:10px 0 18px;font-size:13px;color:var(--muted)}
          .toast{position:fixed;z-index:10;left:50%;bottom:44px;transform:translateX(-50%);opacity:0;transition:opacity .2s;pointer-events:none;font-size:15px;white-space:nowrap}
          .toast.on{opacity:1}
          </style>
          </head>
          <body>
          <div class="wall"></div>
          <div class="stage">
            <header class="top">
              <div class="left">
                <div class="glass pill brand">SALON</div>
                <div class="f glass pill btn" id="btnGlass" tabindex="0" role="button" data-act="glass"><span class="k">Cam</span><span class="v">Hafif · Şeffaf</span></div>
                <div class="f glass pill btn" id="btnRestore" tabindex="0" role="button" data-act="restore" hidden><span class="k">Gizlenenler</span><span class="v" id="hidN">0</span></div>
              </div>
              <div class="glass pill"><span id="time">--:--</span><span id="date"></span></div>
            </header>

            <section class="hero">
              <p class="kicker" id="kicker">Uygulama</p>
              <h1 id="title">Salon</h1>
              <p id="sub"></p>
            </section>

            <div class="scroll" id="scroll"><div class="grid" id="grid"></div></div>

            <div class="hint">Tamam: aç &nbsp;·&nbsp; Tamam’a basılı tut: gizle</div>
          </div>
          <div class="glass pill toast" id="toast"></div>

          <script>
          (function(){
            var A = window.Android || null;
            var root = document.documentElement;
            var grid = document.getElementById('grid');
            var scroller = document.getElementById('scroll');
            var btnGlass = document.getElementById('btnGlass');
            var btnRestore = document.getElementById('btnRestore');
            var hidN = document.getElementById('hidN');
            var toastEl = document.getElementById('toast');
            var el = {k: document.getElementById('kicker'), t: document.getElementById('title'), s: document.getElementById('sub')};

            var PRESETS = [
              {n: 'Hafif · Şeffaf', blur: 0, tint: 0.12},
              {n: 'Hafif · Tonlu', blur: 0, tint: 0.62},
              {n: 'Bulanık · Şeffaf', blur: 1, tint: 0.12},
              {n: 'Bulanık · Tonlu', blur: 1, tint: 0.62}
            ];
            var DEMO = [
              {p: 'demo.video', l: 'Videolar'}, {p: 'demo.tv', l: 'Canlı TV'}, {p: 'demo.music', l: 'Müzik'},
              {p: 'demo.photos', l: 'Fotoğraflar'}, {p: 'demo.games', l: 'Oyunlar'}, {p: 'demo.web', l: 'Tarayıcı'},
              {p: 'demo.files', l: 'Dosyalar'}, {p: 'demo.settings', l: 'Ayarlar'}, {p: 'demo.news', l: 'Haberler'},
              {p: 'demo.movies', l: 'Filmler'}, {p: 'demo.kids', l: 'Çocuk'}, {p: 'demo.sport', l: 'Spor'}
            ];

            var preset = 0, apps = [], demoHidden = [], lastSig = '';
            var pressTimer = null, pressEl = null, held = false, toastTimer = null;

            function getPref(k){
              try { if (A && A.getPref) { return String(A.getPref(k)); } return localStorage.getItem(k) || ''; }
              catch (e) { return ''; }
            }
            function setPref(k, v){
              try { if (A && A.setPref) { A.setPref(k, String(v)); } else { localStorage.setItem(k, String(v)); } }
              catch (e) {}
            }

            function applyPreset(){
              var p = PRESETS[preset];
              if (p.blur) { root.classList.add('blur'); } else { root.classList.remove('blur'); }
              root.style.setProperty('--tint', p.tint);
              btnGlass.querySelector('.v').textContent = p.n;
            }

            function hue(s){
              var h = 0, i;
              for (i = 0; i < s.length; i++) { h = (h * 31 + s.charCodeAt(i)) % 360; }
              return h;
            }

            function toast(msg){
              toastEl.textContent = msg;
              toastEl.className = 'glass pill toast on';
              clearTimeout(toastTimer);
              toastTimer = setTimeout(function(){ toastEl.className = 'glass pill toast'; }, 2800);
            }

            function readApps(){
              if (A && A.apps) {
                try { var raw = String(A.apps()); lastSig = raw; return JSON.parse(raw); }
                catch (e) { return []; }
              }
              lastSig = '';
              return DEMO.filter(function(a){ return demoHidden.indexOf(a.p) < 0; });
            }
            function hiddenCount(){
              try { if (A && A.hiddenCount) { return A.hiddenCount(); } } catch (e) {}
              return demoHidden.length;
            }

            function makeApp(a){
              var d = document.createElement('div');
              d.className = 'f app';
              d.tabIndex = 0;
              d.setAttribute('role', 'button');
              d.setAttribute('data-p', a.p);
              d.setAttribute('data-l', a.l);
              d.style.setProperty('--h', hue(a.p));
              var sq = document.createElement('span');
              sq.className = 'sq';
              var img = new Image();
              img.alt = '';
              img.onerror = function(){
                if (img.parentNode) { sq.removeChild(img); }
                sq.className = 'sq ph';
                sq.appendChild(document.createTextNode(a.l.charAt(0).toUpperCase()));
              };
              img.src = 'icon/' + encodeURIComponent(a.p);
              sq.appendChild(img);
              var lb = document.createElement('span');
              lb.className = 'lbl';
              lb.textContent = a.l;
              d.appendChild(sq);
              d.appendChild(lb);
              d.addEventListener('click', function(){ activate(d); });
              return d;
            }

            function render(focusPkg, focusIdx){
              apps = readApps();
              while (grid.firstChild) { grid.removeChild(grid.firstChild); }
              var i;
              if (!apps.length) {
                var m = document.createElement('div');
                m.className = 'empty';
                m.textContent = 'Gösterilecek uygulama yok.';
                grid.appendChild(m);
              }
              for (i = 0; i < apps.length; i++) { grid.appendChild(makeApp(apps[i])); }
              var n = hiddenCount();
              btnRestore.hidden = n < 1;
              hidN.textContent = n;
              var items = grid.querySelectorAll('.app'), target = null;
              if (focusPkg) {
                for (i = 0; i < items.length; i++) { if (items[i].getAttribute('data-p') === focusPkg) { target = items[i]; } }
              } else if (focusIdx !== null && focusIdx !== undefined && items.length) {
                target = items[Math.min(focusIdx, items.length - 1)];
              }
              if (target) { setFocus(target); }
            }

            function updateHero(x){
              var k, t, s, h;
              if (x.classList.contains('app')) {
                k = 'Uygulama'; t = x.getAttribute('data-l');
                s = 'Açmak için Tamam, gizlemek için Tamam’a basılı tut.';
                h = hue(x.getAttribute('data-p'));
              } else if (x.getAttribute('data-act') === 'glass') {
                k = 'Görünüm'; t = 'Cam: ' + PRESETS[preset].n;
                s = 'Değiştirmek için Tamam. Hafif modlar eski televizyonlarda daha akıcı çalışır.';
                h = 220;
              } else {
                k = 'Gizlenenler'; t = 'Hepsini geri getir';
                s = 'Gizlediğin uygulamaları listeye tekrar ekler.';
                h = 280;
              }
              el.k.textContent = k; el.t.textContent = t; el.s.textContent = s;
              root.style.setProperty('--hue', h);
            }
            document.addEventListener('focusin', function(e){
              var t = e.target;
              if (t && t.classList && t.classList.contains('f')) { updateHero(t); }
            });

            function keepVisible(x){
              if (!scroller.contains(x)) { return; }
              var sr = scroller.getBoundingClientRect(), r = x.getBoundingClientRect(), pad = 28;
              if (r.top < sr.top + pad) { scroller.scrollTop -= (sr.top + pad - r.top); }
              else if (r.bottom > sr.bottom - pad) { scroller.scrollTop += (r.bottom - (sr.bottom - pad)); }
            }
            function setFocus(x){
              try { x.focus({preventScroll: true}); } catch (e) { x.focus(); }
              keepVisible(x);
            }
            function focusFirst(){
              var f = grid.querySelector('.app');
              if (f) { setFocus(f); }
            }

            /* Uzamsal gezinme: yone gore en yakin ogeyi bulur */
            function move(dir){
              var cur = document.activeElement;
              var list = document.querySelectorAll('.f');
              if (!cur || !cur.classList || !cur.classList.contains('f')) { focusFirst(); return; }
              var r = cur.getBoundingClientRect();
              var cx = r.left + r.width / 2, cy = r.top + r.height / 2;
              var best = null, bs = 1e12, i, x, b, dx, dy, p, s, sc;
              for (i = 0; i < list.length; i++) {
                x = list[i];
                if (x === cur || x.offsetParent === null) { continue; }
                b = x.getBoundingClientRect();
                dx = b.left + b.width / 2 - cx;
                dy = b.top + b.height / 2 - cy;
                if (dir === 'L') { if (dx > -8) { continue; } p = -dx; s = Math.abs(dy); }
                else if (dir === 'R') { if (dx < 8) { continue; } p = dx; s = Math.abs(dy); }
                else if (dir === 'U') { if (dy > -8) { continue; } p = -dy; s = Math.abs(dx); }
                else { if (dy < 8) { continue; } p = dy; s = Math.abs(dx); }
                sc = p + s * 3;
                if (sc < bs) { bs = sc; best = x; }
              }
              if (best) { setFocus(best); }
            }

            function activate(x){
              if (x.classList.contains('app')) {
                var p = x.getAttribute('data-p');
                if (A && A.launch) { try { A.launch(p); } catch (e) {} }
                else { toast('Önizleme: ' + x.getAttribute('data-l')); }
                return;
              }
              var act = x.getAttribute('data-act');
              if (act === 'glass') {
                preset = (preset + 1) % PRESETS.length;
                setPref('preset', preset);
                applyPreset();
                updateHero(x);
              } else if (act === 'restore') {
                if (A && A.unhideAll) { try { A.unhideAll(); } catch (e) {} }
                demoHidden = [];
                render(null, null);
                focusFirst();
                toast('Gizlenen uygulamalar geri geldi');
              }
            }

            function hideApp(x){
              var p = x.getAttribute('data-p'), l = x.getAttribute('data-l');
              var list = grid.querySelectorAll('.app'), idx = 0, i;
              for (i = 0; i < list.length; i++) { if (list[i] === x) { idx = i; } }
              if (A && A.hide) { try { A.hide(p); } catch (e) {} } else { demoHidden.push(p); }
              render(null, idx);
              toast('Gizlendi: ' + l + ' · Üstteki Gizlenenler ile geri getirebilirsin');
            }

            function startPress(){
              if (pressEl) { return; }
              var c = document.activeElement;
              if (!c || !c.classList || !c.classList.contains('f')) { return; }
              pressEl = c; held = false;
              if (c.classList.contains('app')) {
                pressTimer = setTimeout(function(){
                  held = true;
                  if (pressEl) { hideApp(pressEl); }
                }, 700);
              }
            }
            function endPress(){
              clearTimeout(pressTimer);
              var t = pressEl;
              pressEl = null;
              if (t && !held) { activate(t); }
              held = false;
            }
            window.addEventListener('blur', function(){ clearTimeout(pressTimer); pressEl = null; held = false; });

            var DIRS = {37: 'L', 39: 'R', 38: 'U', 40: 'D'};
            document.addEventListener('keydown', function(e){
              var k = e.keyCode, dir = DIRS[k];
              if (dir) { e.preventDefault(); move(dir); return; }
              if (k === 13 || k === 23 || e.key === 'Enter') { e.preventDefault(); startPress(); return; }
              if (k === 82 || e.key === 'ContextMenu') {
                e.preventDefault();
                var c = document.activeElement;
                if (c && c.classList && c.classList.contains('app')) { hideApp(c); }
              }
            });
            document.addEventListener('keyup', function(e){
              var k = e.keyCode;
              if (k === 13 || k === 23 || e.key === 'Enter') { e.preventDefault(); endPress(); }
            });

            /* Saat */
            var DAYS = ['Pazar', 'Pazartesi', 'Salı', 'Çarşamba', 'Perşembe', 'Cuma', 'Cumartesi'];
            var MONTHS = ['Ocak', 'Şubat', 'Mart', 'Nisan', 'Mayıs', 'Haziran', 'Temmuz', 'Ağustos', 'Eylül', 'Ekim', 'Kasım', 'Aralık'];
            function pad(n){ return (n < 10 ? '0' : '') + n; }
            function tick(){
              var n = new Date();
              document.getElementById('time').textContent = pad(n.getHours()) + ':' + pad(n.getMinutes());
              document.getElementById('date').textContent = n.getDate() + ' ' + MONTHS[n.getMonth()] + ' ' + DAYS[n.getDay()];
            }

            /* Android tarafindan cagrilir */
            window.salonRefresh = function(){
              if (A && A.apps) {
                var raw = '';
                try { raw = String(A.apps()); } catch (e) { return; }
                if (raw === lastSig) { return; }
              }
              var cur = document.activeElement;
              render((cur && cur.getAttribute) ? cur.getAttribute('data-p') : null, null);
            };
            window.salonHome = function(){ scroller.scrollTop = 0; focusFirst(); };

            var saved = parseInt(getPref('preset'), 10);
            if (saved >= 0 && saved < PRESETS.length) { preset = saved; }
            applyPreset();
            tick();
            setInterval(tick, 20000);
            render(null, null);
            focusFirst();
          })();
          </script>
          </body>
          </html>
          SALON_EOF
      - name: APK derle
        working-directory: salon
        run: gradle assembleDebug --no-daemon --stacktrace
      - name: APK dosyasını hazırla
        run: cp salon/app/build/outputs/apk/debug/app-debug.apk Salon.apk
      - name: Çıktıyı sakla
        uses: actions/upload-artifact@v4
        with:
          name: Salon-APK
          path: Salon.apk
      - name: Releases bölümüne yükle
        uses: softprops/action-gh-release@v2
        with:
          tag_name: latest
          name: Salon Launcher
          body: Android TV için Salon Launcher. Salon.apk dosyasını indirip kur.
          files: Salon.apk
          make_latest: true
