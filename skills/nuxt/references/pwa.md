# Minimal PWA Support for Nuxt

Use this reference whenever a user asks for PWA functionality in a Nuxt app. By default, implement only the minimal setup in this document, not a full offline PWA.

## Goal

Add only the minimal PWA support already used in this project:

- web app manifest
- app icons
- head metadata linking the manifest
- a minimal service worker file
- a client-only plugin that registers the service worker

Do **not** add a full PWA stack unless the user explicitly asks for specific additional PWA features.

## Default Rule

If the user says things like:

- "make this a PWA"
- "add PWA support"
- "enable PWA functionality"

implement only the minimal setup described here unless they explicitly ask for offline support, caching, background sync, push notifications, or other advanced PWA behavior.

## What To Add

### 1. Manifest

Create or update `public/manifest.json` with at least:

- `name`
- `short_name`
- `description`
- `start_url`
- `display: "standalone"`
- `background_color`
- `theme_color`
- `icons`

Example:

```json
{
  "name": "Paper Summary",
  "short_name": "Papers",
  "description": "Upload academic papers and generate AI summaries",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0f1117",
  "theme_color": "#6366f1",
  "icons": [
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    }
  ]
}
```

### 2. Head metadata in `nuxt.config.ts`

Ensure the app head includes:

- manifest link
- theme color
- optional Apple mobile web app tags
- icon links as needed

Typical shape:

```ts
export default defineNuxtConfig({
  app: {
    head: {
      meta: [
        { name: 'theme-color', content: '#6366f1' },
        { name: 'apple-mobile-web-app-capable', content: 'yes' },
        { name: 'apple-mobile-web-app-status-bar-style', content: 'black-translucent' },
        { name: 'apple-mobile-web-app-title', content: 'Papers' }
      ],
      link: [
        { rel: 'manifest', href: '/manifest.json' },
        { rel: 'apple-touch-icon', sizes: '192x192', href: '/icons/icon-192x192.png' }
      ]
    }
  }
})
```

### 3. Minimal service worker

Add `public/sw.js` with no caching logic.

Use this exact minimal pattern:

```js
self.addEventListener('install', () => {
  self.skipWaiting()
})

self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim())
})

// Intentionally no caching/offline logic.
// Kept minimal for installability support.
self.addEventListener('fetch', () => {})
```

This keeps the app aligned with minimal PWA support without implementing offline behavior.

### 4. Register the service worker on the client

Add a client-only plugin such as `app/plugins/sw.client.ts`:

```ts
export default defineNuxtPlugin(() => {
  if (!('serviceWorker' in navigator)) {
    return
  }

  const register = async () => {
    try {
      await navigator.serviceWorker.register('/sw.js', { scope: '/' })
    } catch (error) {
      console.error('Service worker registration failed:', error)
    }
  }

  if (document.readyState === 'complete') {
    void register()
  } else {
    window.addEventListener(
      'load',
      () => {
        void register()
      },
      { once: true }
    )
  }
})
```

## What Not To Add By Default

Unless the user explicitly asks for full PWA capabilities, do **not** add:

- `vite-pwa`
- `@vite-pwa/nuxt`
- Workbox configuration
- precaching
- runtime caching
- offline fallback pages
- push notifications
- background sync
- install prompt UI helpers
- aggressive asset caching strategies

## Why This Is The Default

This setup keeps the Nuxt app simple while providing the minimal PWA functionality used in this project. It avoids the complexity and behavior changes of offline caching and other advanced PWA features.

## Verification

Ask the user to verify locally in the browser:

1. Open DevTools → `Application`
2. Check `Manifest`
3. Check `Service Workers`
4. Confirm `/sw.js` is registered

## Deployment Note

Installability behavior typically requires HTTPS in production. `localhost` is acceptable for development.
