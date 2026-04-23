# PWA

Make a Nuxt app installable as a PWA with a minimal service worker.

## Web App Manifest

Place `manifest.json` in `public/manifest.json`:

```json
{
  "name": "App Name",
  "short_name": "Short",
  "description": "App description",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0f1117",
  "theme_color": "#6366f1",
  "icons": [
    { "src": "/icons/icon-192x192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icons/icon-512x512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    { "src": "/icons/icon.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any" }
  ]
}
```

## Service Worker

Place a minimal service worker in `public/sw.js`:

```js
self.addEventListener('install', () => {
  self.skipWaiting()
})

self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim())
})

self.addEventListener('fetch', () => {})
```

Intentionally omit caching/offline logic if installability support is the only goal.

## Registration Plugin

Register the service worker from a client-only plugin at `app/plugins/sw.client.ts`:

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
    window.addEventListener('load', () => { void register() }, { once: true })
  }
})
```

## Head Meta Tags

Link the manifest and add PWA meta tags in `nuxt.config.ts` under `app.head`:

```ts
app: {
  head: {
    meta: [
      { name: 'theme-color', content: '#6366f1' },
      { name: 'apple-mobile-web-app-capable', content: 'yes' },
      { name: 'apple-mobile-web-app-status-bar-style', content: 'black-translucent' },
      { name: 'apple-mobile-web-app-title', content: 'Short Name' }
    ],
    link: [
      { rel: 'manifest', href: '/manifest.json' },
      { rel: 'apple-touch-icon', sizes: '192x192', href: '/icons/icon-192x192.png' }
    ]
  }
}
```

## Icons

Place icon files in `public/icons/`:
- `icon-192x192.png`
- `icon-512x512.png`
- `icon.svg`
