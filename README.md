````md
# 🔒 DevTools Detection in Nuxt 3

This project includes a complete setup to **detect browser DevTools usage** in a Nuxt 3 application. When DevTools are opened, users will be redirected to a restricted error page (`/devtools-detected`), helping to prevent potential client-side manipulation or inspection.

---

## 📦 Features

- 🚫 Detects DevTools opening in real-time
- ⚙️ Works only in `STAGING` and `PRODUCTION` environments
- 🍪 Uses cookies to track detection state
- ⛔ Redirects detected users to a 403 error page
- 🧼 Automatically clears detection cookie after use

---

## ⚙️ Environment Setup
1. Create a `.env` file in the project root: NUXT_ENVIRONMENT=LOCAL
2. Supported values:

   * `LOCAL` – DevTools detection disabled
   * `STAGING` – Detection enabled
   * `PRODUCTION` – Detection enabled

---

## 🔧 Configuration

### 1. Runtime Environment

Update `nuxt.config.ts` to expose the environment variable:

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      environment: process.env.NUXT_ENVIRONMENT,
    },
  },
});
```

---

### 2. DevTools Detection Plugin

Create `plugins/detect-devtools.client.ts`:

```ts
export default defineNuxtPlugin(() => {
  if (import.meta.server) return;

  const config = useRuntimeConfig();
  const env = config.public.environment;
  if (env === 'LOCAL') return;

  const cookie = useCookie('devtools-detected');

  // 1. Early detection script
  const earlyScript = document.createElement('script');
  earlyScript.innerHTML = `
    const start = performance.now();
    debugger;
    const end = performance.now();
    if (end - start > 100) {
      document.cookie = 'devtools-detected=true; path=/';
      window.location.href = '/devtools-detected';
    }
  `;
  document.head.appendChild(earlyScript);

  // 2. Runtime detection
  const detectViaDebugger = () => {
    const start = performance.now();
    debugger;
    const end = performance.now();
    if (end - start > 100) {
      cookie.value = 'true';
      window.location.href = '/devtools-detected';
    }
  };

  // 3. Detection intervals
  setTimeout(() => detectViaDebugger(), 300);
  const interval = setInterval(() => detectViaDebugger(), 1000);
  window.addEventListener('beforeunload', () => clearInterval(interval));
});
```

---

### 3. Middleware for DevTools Page Access

Create `middleware/devtools.ts`:

```ts
export default defineNuxtRouteMiddleware(() => {
  const cookie = useCookie('devtools-detected');

  if (!cookie.value) {
    return navigateTo('/');
  }

  // Clear cookie after access
  cookie.value = '';
});
```

---

### 4. Error Page

Create `pages/devtools-detected.vue`:

```vue
<script setup lang="ts">
definePageMeta({
  middleware: ['devtools'],
});

throw createError({
  statusCode: 403,
  statusMessage: 'DevTools Detected!',
});
</script>

<template>
  <div></div>
</template>
```

---

## 🔁 Flow Summary

1. User opens the app.
2. Plugin checks if DevTools are open using `debugger` trick.
3. If detected, set a cookie and redirect to `/devtools-detected`.
4. Middleware protects that page and ensures proper access.
5. User sees a 403 error, and cookie is cleared to reset state.

---

## 🧪 Testing

To test detection:

1. Set `NUXT_ENVIRONMENT=STAGING`
2. Run the app
3. Open browser DevTools
4. You should be redirected to `/devtools-detected` and receive a 403 error
