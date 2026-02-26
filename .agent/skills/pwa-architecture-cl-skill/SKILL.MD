# Skill: Progressive Web App (PWA) Architecture & Management

> **Internal Engineering Handbook** — Production-grade reference. No fluff. Assume you're building at scale.

---

## 1. Core Definition

### What is a Progressive Web App?

A PWA is a web application that uses modern browser APIs and progressive enhancement to deliver app-like experiences: installability, offline support, push notifications, and OS-level integration — without an app store.

**Key distinction from SPA:** A SPA is a rendering pattern. A PWA is a deployment and capability contract. A SPA can be a PWA. A PWA doesn't have to be a SPA.

**Key distinction from native:** PWAs run in a browser engine sandbox. They cannot access the full OS API surface. Platform limitations are real and must be documented (see Section 5).

### The 3 Technical Pillars

| Pillar | Role | Minimum Requirement |
|---|---|---|
| **HTTPS** | Security foundation, required by browsers for SW registration | Valid TLS cert (localhost exempt for dev) |
| **Web App Manifest** | Tells the browser this is installable, defines appearance | `name`, `icons`, `start_url`, `display` |
| **Service Worker** | Enables offline, background sync, push, caching | Must be registered and controlling the page |

### Installability Criteria

**Chrome/Edge (Desktop & Android):**
- Served over HTTPS
- Has a Web App Manifest with: `name` or `short_name`, `icons` (192px + 512px), `start_url`, `display` (standalone/fullscreen/minimal-ui)
- Has a registered service worker with a fetch handler
- Not already installed
- Heuristic engagement score (Chrome may suppress prompt for low-engagement sites)

**Safari (iOS 16.4+ for push, iOS 14+ for basic install):**
- Add to Home Screen is manual — no install prompt API
- `apple-mobile-web-app-capable` meta tag required for standalone mode
- No `beforeinstallprompt` event — ever
- Push requires iOS 16.4+ and user must have added to home screen first

**Firefox:** Manifest + SW required. No install prompt on desktop Firefox as of 2024.

---

## 2. Architecture Deep Dive

### 2.1 Application Shell Model

The App Shell separates the minimal UI chrome (nav, layout) from content. The shell is cached aggressively; content is fetched dynamically.

**Use when:**
- App has a persistent navigation or layout wrapper
- Content changes frequently but chrome doesn't
- You need sub-second repeat visits

**Don't use when:**
- Content IS the shell (document-centric apps, blogs)
- SSR is required for first-meaningful-paint SEO
- You're building a multi-page app without client-side routing

**Tradeoff:**
```
First load: slower (shell + content both downloaded)
Repeat load: near-instant (shell from cache, content fetched)
SEO: poor unless you layer SSR or prerendering
```

### 2.2 Rendering Strategies

| Strategy | Description | SEO | Installability | Recommended Stack |
|---|---|---|---|---|
| **CSR** | Browser renders all | Poor (crawler must execute JS) | ✅ | React/Vue + Vite |
| **SSR** | Server renders on request | ✅ | ✅ | Next.js, Nuxt, SvelteKit |
| **SSG** | Pre-rendered at build time | ✅✅ | ✅ | Astro, Next.js static, Eleventy |
| **ISR** | SSG with revalidation window | ✅✅ | ✅ | Next.js ISR, Netlify DPR |

**Recommendation for PWAs:**
- Public-facing + SEO-critical → **SSG or ISR** with client hydration
- App-like (authenticated, dynamic) → **SSR or CSR** with shell caching
- Hybrid → SSR shell, CSR content (SvelteKit, Next.js app router)

### 2.3 Offline & Caching Strategies

#### Cache-First
```
Request → Cache? → Return cached → Background fetch to update
```
Best for: Static assets, fonts, icons, versioned bundles.  
Risk: Stale content if cache is not invalidated properly.

#### Network-First
```
Request → Network? → Return response + update cache → Fallback to cache
```
Best for: API responses, user-specific data.  
Risk: Slow on poor connections (full network timeout before fallback).

#### Stale-While-Revalidate
```
Request → Return cache immediately → Fetch network in background → Update cache
```
Best for: Content that can tolerate being slightly stale (news feeds, product listings).  
Sweet spot for most app content.

#### Background Sync
```
Failed mutation → Queue in IndexedDB → SW retries when online
```
Best for: Form submissions, analytics events, offline-first write operations.  
⚠️ **iOS limitation:** Not supported on Safari. Must implement manual queue with `online` event listener as fallback.

#### Strategy Decision Tree

```
Is this a static asset (JS/CSS/font/image)?
  └─ YES → Cache-First with versioned cache key

Is this an API call?
  ├─ Read-only, tolerate staleness? → Stale-While-Revalidate
  ├─ Must be fresh? → Network-First with cache fallback
  └─ Write/mutation? → Queue for Background Sync

Is this an HTML navigation request?
  ├─ App Shell model? → Cache shell, Network-First for content
  └─ MPA? → Network-First with offline fallback page
```

---

## 3. Service Worker Engineering

### Lifecycle

```
           Registration
                │
           Install event
         (cache static assets)
                │
           Activate event
         (clean old caches)
                │
           Controlling pages
         (intercept fetch events)
```

### Vanilla JS Service Worker — Production Template

```javascript
const CACHE_VERSION = 'v3';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;

const STATIC_ASSETS = [
  '/',
  '/offline.html',
  '/app.css',
  '/app.js',
  '/icons/icon-192.png',
];

// INSTALL: pre-cache static shell
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then(cache => cache.addAll(STATIC_ASSETS))
  );
  // Force activate immediately — use with caution (see update notes)
  self.skipWaiting();
});

// ACTIVATE: purge stale caches
self.addEventListener('activate', event => {
  const allowedCaches = [STATIC_CACHE, DYNAMIC_CACHE];
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(
        keys
          .filter(key => !allowedCaches.includes(key))
          .map(key => caches.delete(key))
      )
    ).then(() => self.clients.claim())
  );
});

// FETCH: stale-while-revalidate for API, cache-first for static
self.addEventListener('fetch', event => {
  const { request } = event;
  const url = new URL(request.url);

  // Skip non-GET and cross-origin
  if (request.method !== 'GET' || url.origin !== location.origin) return;

  // Static assets: cache-first
  if (STATIC_ASSETS.includes(url.pathname)) {
    event.respondWith(caches.match(request).then(cached => cached || fetch(request)));
    return;
  }

  // API: network-first with cache fallback
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(
      fetch(request)
        .then(response => {
          const clone = response.clone();
          caches.open(DYNAMIC_CACHE).then(cache => cache.put(request, clone));
          return response;
        })
        .catch(() => caches.match(request).then(cached => cached || caches.match('/offline.html')))
    );
    return;
  }

  // Navigation: network-first, fallback to offline page
  if (request.mode === 'navigate') {
    event.respondWith(
      fetch(request).catch(() => caches.match('/offline.html'))
    );
    return;
  }
});
```

### Update Strategy Patterns

**⚠️ The `skipWaiting()` trap:** Calling `skipWaiting()` + `clients.claim()` means new SW activates while old pages still run. If the SW serves different assets than those pages expect, you get broken clients mid-session.

**Safe patterns:**

1. **Prompt-to-reload (recommended for apps):**
```javascript
// In SW
self.addEventListener('install', event => {
  // DO NOT call skipWaiting() here
});

// In app code
navigator.serviceWorker.addEventListener('controllerchange', () => {
  window.location.reload(); // Only fires after user acknowledges update
});

// Detect waiting SW and prompt user
const reg = await navigator.serviceWorker.getRegistration();
if (reg?.waiting) {
  showUpdateBanner(() => reg.waiting.postMessage({ type: 'SKIP_WAITING' }));
}
```

2. **Auto-update for static sites (low risk):** Use `skipWaiting()` + `clients.claim()` freely when content is purely static and there's no app state to corrupt.

### Cache Versioning

Always version caches with a build hash or semantic version:
```javascript
const CACHE_VERSION = process.env.BUILD_HASH || 'dev';
```
Never use timestamps — they prevent cache hits across identical builds.

### Avoiding Infinite Update Loops

- Don't fetch the SW script itself inside the SW fetch handler
- Don't cache the SW registration page without careful exclusion
- Ensure `updateViaCache: 'none'` in SW registration:
```javascript
navigator.serviceWorker.register('/sw.js', { updateViaCache: 'none' });
```

### Debugging

```bash
# Chrome DevTools
Application → Service Workers → check "Update on reload" for dev
Application → Cache Storage → inspect cache contents
Application → Background Sync → inspect queues

# Force SW update in console
navigator.serviceWorker.getRegistration().then(r => r.update())
```

### Security Considerations

- A SW controls all fetch requests on its scope. **XSS → SW injection = full page control.** Strong CSP is non-negotiable.
- Limit SW scope to minimum necessary path
- Never store secrets in SW or Cache API — they're accessible to any script in scope
- Validate all responses before caching (check `response.ok`, response type)

---

## 4. Web App Manifest Deep Dive

### Minimum Viable Manifest

```json
{
  "name": "My Application",
  "short_name": "MyApp",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#1a73e8",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-512-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

### Advanced Fields

**`display_override`** — Ordered list of display modes, browser picks first supported:
```json
"display_override": ["window-controls-overlay", "standalone", "browser"]
```

**`shortcuts`** — Quick action menu (long-press on Android, right-click on desktop):
```json
"shortcuts": [
  {
    "name": "New Document",
    "short_name": "New",
    "url": "/new",
    "icons": [{ "src": "/icons/new.png", "sizes": "96x96" }]
  }
]
```

**`share_target`** — Receive shares from other apps:
```json
"share_target": {
  "action": "/share-handler",
  "method": "POST",
  "enctype": "multipart/form-data",
  "params": {
    "title": "title",
    "text": "text",
    "url": "url",
    "files": [{ "name": "media", "accept": ["image/*"] }]
  }
}
```

**`protocol_handlers`** — Register as handler for custom protocols:
```json
"protocol_handlers": [
  { "protocol": "web+myapp", "url": "/handle?uri=%s" }
]
```

**`file_handlers`** — Open files of specific types from OS:
```json
"file_handlers": [
  {
    "action": "/open-file",
    "accept": { "text/csv": [".csv"] }
  }
]
```

### iOS-Specific Meta Tags

```html
<!-- Required for standalone mode on iOS -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="MyApp">

<!-- iOS icons (manifest icons ignored on older iOS) -->
<link rel="apple-touch-icon" sizes="180x180" href="/icons/apple-180.png">
<link rel="apple-touch-icon" sizes="152x152" href="/icons/apple-152.png">
<link rel="apple-touch-icon" sizes="120x120" href="/icons/apple-120.png">

<!-- iOS splash screens — must be exact device dimensions or ignored -->
<link rel="apple-touch-startup-image" href="/splash/iphone14pro.png"
  media="(device-width: 390px) and (device-height: 844px) and (-webkit-device-pixel-ratio: 3)">
```

⚠️ **iOS splash screens:** Apple requires pixel-perfect splash images per device. Use a generator (e.g., `pwa-asset-generator`) to automate. Missing splash = white flash on load.

### Maskable Icons

Maskable icons fill the full icon container on Android (no white padding). The "safe zone" is the central 80% circle.

```
Total icon: 512x512
Safe zone:  ~409x409 (80% circle)
Keep all meaningful content within safe zone
```

Generate with: https://maskable.app

---

## 5. Platform Limitations ⚠️

### iOS / Safari

| Feature | Status | Notes |
|---|---|---|
| Install prompt API | ❌ Never | Manual Add to Home Screen only |
| Push notifications | ⚠️ iOS 16.4+ only | Must be in standalone mode, user must grant |
| Background sync | ❌ Not supported | Implement fallback with `online` event |
| Periodic background sync | ❌ Not supported | No workaround |
| Web Share Target | ❌ Not supported | |
| File System Access API | ❌ Not supported | |
| Storage persistence | ❌ No `persist()` API | Safari evicts storage aggressively under pressure |
| IndexedDB stability | ⚠️ Bugs in older iOS | Cross-tab IDB transactions can deadlock on older WebKit |
| Payment Request | ⚠️ Partial | Apple Pay via Payment Request works |
| SW update behavior | ⚠️ Quirky | SW may not update predictably after iOS upgrades |
| Push payload size | ~4KB | Same as Chrome |

**Critical iOS gotcha:** Safari will **evict all PWA data** (IndexedDB, Cache API, SW registrations) if the PWA hasn't been opened in ~7 days and device is under storage pressure. There is no API to prevent this. Design accordingly — treat iOS storage as ephemeral.

### Android / Chrome

| Feature | Status | Notes |
|---|---|---|
| Install prompt | ✅ Full | `beforeinstallprompt` + Play Store via TWA |
| Push notifications | ✅ Full | FCM-backed |
| Background sync | ✅ Full | |
| Periodic background sync | ✅ | Requires granted permission |
| File System Access | ✅ | Full OPFS + showOpenFilePicker |
| Storage persistence | ✅ | `navigator.storage.persist()` supported |
| Share Target | ✅ Full | |

### Desktop (Chrome/Edge)

| Feature | Status | Notes |
|---|---|---|
| Installation | ✅ | Creates OS shortcut/app entry |
| Window Controls Overlay | ✅ Chrome/Edge | Custom title bar |
| Protocol handlers | ✅ Chrome | Register with OS |
| File handlers | ✅ Chrome/Edge | Open files from OS |
| Push notifications | ✅ | Uses OS notification system |
| Badge API | ✅ Chrome/Edge | App icon badging |

**Desktop install reality:** Users rarely discover the install button. Consider explicitly promoting it. Desktop PWA UX is still maturing — test thoroughly on Windows, macOS, Linux.

---

## 6. Push Notifications

### Protocol Overview

Web Push uses the VAPID (Voluntary Application Server Identification) protocol:

```
App Server → Web Push Service (FCM/APNs) → Browser → SW push event → Notification
```

### VAPID Key Generation

```bash
npx web-push generate-vapid-keys
# Outputs: Public key + Private key
# Store private key server-side ONLY, never expose
```

### Client-Side Subscription

```javascript
async function subscribeToPush() {
  const reg = await navigator.serviceWorker.ready;
  const subscription = await reg.pushManager.subscribe({
    userVisibleOnly: true, // Required — silent pushes not allowed
    applicationServerKey: urlBase64ToUint8Array(PUBLIC_VAPID_KEY)
  });
  // Send subscription to your server
  await fetch('/api/push-subscribe', {
    method: 'POST',
    body: JSON.stringify(subscription),
    headers: { 'Content-Type': 'application/json' }
  });
}

function urlBase64ToUint8Array(base64String) {
  const padding = '='.repeat((4 - base64String.length % 4) % 4);
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/');
  const rawData = atob(base64);
  return new Uint8Array([...rawData].map(c => c.charCodeAt(0)));
}
```

### Service Worker Push Handler

```javascript
self.addEventListener('push', event => {
  const data = event.data?.json() ?? { title: 'Update', body: '' };
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icons/icon-192.png',
      badge: '/icons/badge-72.png',
      data: { url: data.url },
      actions: data.actions || []
    })
  );
});

self.addEventListener('notificationclick', event => {
  event.notification.close();
  const url = event.notification.data?.url || '/';
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then(clientList => {
      const existing = clientList.find(c => c.url === url && 'focus' in c);
      return existing ? existing.focus() : clients.openWindow(url);
    })
  );
});
```

### Server-Side (Node.js with web-push)

```javascript
import webpush from 'web-push';

webpush.setVapidDetails(
  'mailto:you@example.com',
  process.env.VAPID_PUBLIC_KEY,
  process.env.VAPID_PRIVATE_KEY
);

await webpush.sendNotification(
  subscription, // From client subscription object
  JSON.stringify({ title: 'Hello', body: 'World', url: '/dashboard' })
);
```

### Browser Support Matrix

| Browser | Push Support | VAPID | Notes |
|---|---|---|---|
| Chrome (Android/Desktop) | ✅ | ✅ | Via FCM |
| Edge | ✅ | ✅ | Via WNS + FCM |
| Firefox | ✅ | ✅ | Via Mozilla Push |
| Safari macOS 13+ | ✅ | ✅ | APNs-backed |
| Safari iOS 16.4+ | ✅ | ✅ | Must be installed to Home Screen |
| Safari iOS < 16.4 | ❌ | ❌ | No workaround |

---

## 7. Storage Strategies

### Storage Decision Matrix

| Storage | Capacity | Persistence | Sync | Best For |
|---|---|---|---|---|
| `localStorage` | ~5-10MB | Until cleared | Sync (blocks UI) | Small config, auth tokens |
| `sessionStorage` | ~5-10MB | Tab lifetime | Sync | Temp session data |
| `IndexedDB` | Quota-based | Until cleared | Async | Large datasets, offline DB |
| `Cache API` | Quota-based | Until evicted | Async | HTTP response caching |
| `OPFS` | Quota-based | Persistent | Async | Large files, SQLite |
| `Cookies` | ~4KB | Configurable | Sync (per request) | Auth, server-read data |

### Storage Quotas

```
Chrome:  ~60% of available disk space (shared across Cache + IDB + OPFS)
Firefox: ~50% of disk space
Safari:  ~1GB per origin, aggressive eviction after ~7 days inactivity

Check current usage:
const estimate = await navigator.storage.estimate();
console.log(estimate.quota, estimate.usage);
```

### Request Persistent Storage

```javascript
// Without this, browser may evict data at will
const isPersisted = await navigator.storage.persisted();
if (!isPersisted) {
  const granted = await navigator.storage.persist();
  // May require user gesture or bookmark on some browsers
}
```

### IndexedDB Pattern (with error handling)

```javascript
function openDB(name, version, upgradeCallback) {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(name, version);
    req.onupgradeneeded = e => upgradeCallback(e.target.result);
    req.onsuccess = e => resolve(e.target.result);
    req.onerror = e => reject(e.target.error);
    req.onblocked = () => console.warn('IDB blocked — another tab may need to close');
  });
}
```

⚠️ **iOS IDB Warning:** On iOS < 14, IDB transactions can hang indefinitely if multiple tabs compete. Always handle `onblocked` and implement timeouts.

---

## 8. Performance Engineering

### Lighthouse PWA Scoring

PWA audit checks (via Lighthouse or PageSpeed Insights):
- ✅ Registers a SW
- ✅ Responds with 200 when offline
- ✅ Has a valid manifest with required fields
- ✅ Icons are correct sizes
- ✅ start_url responds while offline
- ✅ maskable icon present
- ✅ splash screen configured
- ✅ Sets theme-color meta

### Core Web Vitals Impact

| Metric | PWA-Specific Risk | Mitigation |
|---|---|---|
| **LCP** | First load is slow if shell is large | Inline critical CSS, use SSR for first paint |
| **FID/INP** | SW registration blocks main thread briefly | Register SW after page load |
| **CLS** | Cached assets served before fonts load | `font-display: swap`, skeleton screens |

```javascript
// Register SW after load to not delay FID
window.addEventListener('load', () => {
  navigator.serviceWorker.register('/sw.js', { updateViaCache: 'none' });
});
```

### Preloading & Prefetching

```html
<!-- Critical path resources -->
<link rel="preload" href="/app.css" as="style">
<link rel="preload" href="/fonts/main.woff2" as="font" crossorigin>

<!-- Next navigation -->
<link rel="prefetch" href="/dashboard.js">
<link rel="prefetch" href="/api/user-data" as="fetch" crossorigin>
```

### Bundle Splitting

```javascript
// Route-based code splitting (Vite/Rollup)
const Dashboard = lazy(() => import('./Dashboard'));

// Preload on hover/intent
element.addEventListener('mouseenter', () => import('./Dashboard'));
```

### HTTP Caching Headers

| Asset Type | Cache-Control |
|---|---|
| Hashed JS/CSS bundles | `public, max-age=31536000, immutable` |
| HTML documents | `no-cache` (revalidate every time) |
| Service worker script | `no-cache` or `max-age=0` |
| API responses | `no-store` or short `max-age` |
| Images (hashed) | `public, max-age=31536000, immutable` |

⚠️ **Never cache the SW file.** The browser checks for SW updates every 24h maximum, but setting `Cache-Control: max-age=86400` on `sw.js` means bug fixes won't propagate. Set `max-age=0` or `no-cache`.

---

## 9. Deployment & CI/CD

### HTTPS Setup

- Use Let's Encrypt for free TLS
- Enforce HTTPS at CDN/load balancer level
- Set `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
- SW will not register over HTTP (except localhost)

### CDN Considerations

- Deploy SW file from **same origin** — cross-origin SW not allowed
- Set appropriate `scope` in SW registration (defaults to SW file directory)
- Cache hashed assets at CDN edge; set short TTL or `no-cache` for HTML and SW
- Ensure CDN does not modify SW response (breaks integrity)

### Cache Invalidation Strategy

```
Build pipeline:
  1. Hash all JS/CSS/image filenames
  2. Deploy new files to CDN (old hashes still serve existing users)
  3. Deploy new HTML referencing new hashes
  4. Update SW with new CACHE_VERSION → triggers SW update cycle
  5. Old cached assets expire naturally or are evicted by next SW activation
```

### Service Worker Update Rollout

```
Deploy new SW → Browser detects change on next visit/registration check
→ New SW installs (waits in "waiting" state)
→ User closes all tabs OR accepts prompt → New SW activates
→ Old caches purged in activate event
→ Clients now served by new SW
```

**Gradual rollout:** Route traffic to new SW deployment via feature flags on the HTML page (serve different `sw.js` URLs to different user segments).

---

## 10. SEO & Discoverability

### Essential Meta Tags

```html
<meta name="description" content="...">
<meta property="og:title" content="...">
<meta property="og:description" content="...">
<meta property="og:image" content="https://...">
<meta name="twitter:card" content="summary_large_image">
<link rel="canonical" href="https://...">
<meta name="theme-color" content="#1a73e8">
```

### Structured Data

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "WebApplication",
  "name": "My App",
  "url": "https://myapp.com",
  "applicationCategory": "Productivity"
}
</script>
```

### Dynamic Rendering

For CSR-heavy PWAs where Googlebot might not execute JS reliably:
- Use **SSR or SSG** for public-facing pages
- Or use **dynamic rendering**: detect bot UA server-side, serve pre-rendered HTML
- Tools: Rendertron, Prerender.io, Vercel Edge Middleware

### Sitemap

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://myapp.com/</loc>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
</urlset>
```

---

## 11. App Store Distribution

### Trusted Web Activity (Android)

TWA wraps a PWA in an Android app shell using Chrome Custom Tabs. The web content must be verified via Digital Asset Links.

```bash
# bubblewrap CLI
npm install -g @bubblewrap/cli
bubblewrap init --manifest https://myapp.com/manifest.json
bubblewrap build
# → Produces signed APK for Play Store submission
```

**Verification file required at:**
```
https://myapp.com/.well-known/assetlinks.json

[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.myapp",
    "sha256_cert_fingerprints": ["..."]
  }
}]
```

### iOS App Store

No official PWA → App Store path. Options:

| Approach | Tools | Effort | Risk |
|---|---|---|---|
| WKWebView wrapper | Capacitor, Ionic | Medium | App rejection if thin wrapper |
| React Native WebView | RN + WebView | Medium | More control |
| Full native rewrite | Swift/Kotlin | High | Most stable |

⚠️ **Apple rejection risk:** Apps that are "web clones" with no native value-add get rejected. Add native features (biometrics, widgets, Siri Shortcuts) to justify App Store presence.

---

## 12. Security

### CSP for PWAs

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.myapp.com wss://ws.myapp.com;
  worker-src 'self';
  manifest-src 'self';
  frame-ancestors 'none';
```

⚠️ `unsafe-inline` for scripts defeats XSS protection. Use nonces or hashes.  
⚠️ `worker-src 'self'` is required for SW to register under strict CSP.

### XSS → Service Worker Escalation

If an attacker injects a script that calls `navigator.serviceWorker.register('/evil-sw.js')`:
- They can intercept all requests, steal auth tokens, serve phishing content
- **Mitigation:** Strong CSP, never `eval()` user content, sanitize all DOM insertion

### Secure Storage Patterns

```
❌ Never store:  JWT tokens in localStorage (XSS accessible)
✅ Better:       HttpOnly cookies for session tokens
✅ Acceptable:   Non-sensitive preferences in localStorage
✅ For sensitive data: Encrypt before storing in IDB (use WebCrypto API)
```

### WebCrypto for Storage Encryption

```javascript
const key = await crypto.subtle.generateKey(
  { name: 'AES-GCM', length: 256 }, true, ['encrypt', 'decrypt']
);
const iv = crypto.getRandomValues(new Uint8Array(12));
const encrypted = await crypto.subtle.encrypt(
  { name: 'AES-GCM', iv },
  key,
  new TextEncoder().encode(sensitiveData)
);
```

---

## 13. Maintenance & Monitoring

### Service Worker Error Logging

```javascript
// In SW
self.addEventListener('error', event => {
  // Send to logging endpoint — note: fetch may be intercepted by this SW
  fetch('/api/log', {
    method: 'POST',
    body: JSON.stringify({ type: 'sw-error', message: event.message, stack: event.filename }),
    keepalive: true // Ensures delivery even if SW is about to die
  });
});

self.addEventListener('unhandledrejection', event => {
  fetch('/api/log', {
    method: 'POST',
    body: JSON.stringify({ type: 'sw-unhandled-rejection', reason: String(event.reason) }),
    keepalive: true
  });
});
```

### Offline Analytics Queue

```javascript
// Queue analytics events when offline, flush when online
const ANALYTICS_QUEUE = 'analytics-queue';

async function trackEvent(event) {
  if (!navigator.onLine) {
    const db = await openDB('analytics', 1, db => db.createObjectStore(ANALYTICS_QUEUE, { autoIncrement: true }));
    const tx = db.transaction(ANALYTICS_QUEUE, 'readwrite');
    tx.objectStore(ANALYTICS_QUEUE).add({ ...event, timestamp: Date.now() });
    return;
  }
  await sendAnalytics(event);
}

window.addEventListener('online', flushAnalyticsQueue);
```

### Version Rollback Strategy

```
SW versions: v1 → v2 (broken) → v3 (fix)

Emergency rollback:
  1. Redeploy v1 as sw.js
  2. Bump CACHE_VERSION to force cache purge
  3. Browser detects changed SW → installs → activates
  4. Broken v2 users get v1 on next visit/tab close

Prevention:
  - Canary deployments (serve new SW to 5% of users)
  - Smoke test SW in staging with real-device testing
  - Monitor SW error rates via logging
```

### Feature Detection Patterns

```javascript
// Never assume — always detect
const supportsSW = 'serviceWorker' in navigator;
const supportsPush = 'PushManager' in window;
const supportsPeriodicSync = 'periodicSync' in ServiceWorkerRegistration.prototype;
const supportsBackgroundSync = 'sync' in ServiceWorkerRegistration.prototype;
const supportsInstallPrompt = 'onbeforeinstallprompt' in window; // Chrome only
const supportsBadge = 'setAppBadge' in navigator;
const supportsFileHandlers = 'launchQueue' in window;
```

---

## 14. Anti-Patterns

### Over-Caching
**Problem:** Caching everything including API responses aggressively.  
**Result:** Users see stale data, auth failures, broken state.  
**Fix:** Only cache what makes sense offline. API mutations should never be cached.

### Cache Poisoning
**Problem:** Caching error responses (404, 500) from the network.  
**Result:** Errors served from cache indefinitely.  
**Fix:**
```javascript
fetch(request).then(response => {
  if (!response.ok) return response; // Don't cache errors
  cache.put(request, response.clone());
  return response;
});
```

### Forcing Update Reload Loops
**Problem:** Calling `location.reload()` in `controllerchange` without debounce or guard.  
**Result:** Infinite reload loop if SW keeps updating.  
**Fix:** Only reload once, track with a flag:
```javascript
let refreshing = false;
navigator.serviceWorker.addEventListener('controllerchange', () => {
  if (refreshing) return;
  refreshing = true;
  window.location.reload();
});
```

### Treating PWA as Native Replacement
**Problem:** Assuming PWA can fully replace a native app, especially on iOS.  
**Result:** Missing features (background sync, push), unexpected storage eviction, poor UX on edge cases.  
**Fix:** Explicitly document and design around platform gaps. Use progressive enhancement — degrade gracefully when APIs are absent.

### Registering SW on Every Render
**Problem:** Calling `register()` on every React render or without checking support.  
**Result:** Performance overhead, multiple registration attempts.  
**Fix:** Register once, on `load` event, with feature detection.

### Hardcoding SW Scope to `/`
**Problem:** SW registered at root controls all paths including unrelated subdomains/paths.  
**Fix:** Scope SW to only the paths it needs to control:
```javascript
navigator.serviceWorker.register('/app/sw.js', { scope: '/app/' });
```

---

## 15. Documentation References

| Resource | URL | Notes |
|---|---|---|
| MDN Service Worker API | https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API | Most comprehensive reference |
| MDN Web App Manifest | https://developer.mozilla.org/en-US/docs/Web/Manifest | Field-by-field reference |
| web.dev PWA | https://web.dev/progressive-web-apps/ | Google's opinionated guide |
| W3C Manifest Spec | https://www.w3.org/TR/appmanifest/ | Canonical spec |
| W3C Service Worker Spec | https://www.w3.org/TR/service-workers/ | Canonical spec |
| Web Push Protocol (RFC 8030) | https://datatracker.ietf.org/doc/html/rfc8030 | IETF spec |
| VAPID Spec (RFC 8292) | https://datatracker.ietf.org/doc/html/rfc8292 | IETF spec |
| WebKit Blog | https://webkit.org/blog/ | Track Safari/iOS PWA support changes here |
| Chrome PWA Updates | https://developer.chrome.com/docs/capabilities/ | Project Fugu / new APIs |
| Workbox | https://developer.chrome.com/docs/workbox/ | Production SW library by Google |
| PWA Stats | https://www.pwastats.com/ | Real-world PWA case studies |
| Can I Use — Service Workers | https://caniuse.com/serviceworkers | Browser support table |
| maskable.app | https://maskable.app | Test/preview maskable icons |
| Bubblewrap (TWA) | https://github.com/GoogleChromeLabs/bubblewrap | CLI for Play Store packaging |

---

*Last updated: 2025. Verify Safari/iOS support status at webkit.org/blog before shipping push or background sync features — this changes with major iOS releases.*