# Vite Configuration

Full Vite config for a Vue 3 SPA project. Single config file, ESM format.

## vite.config.js

```js
import vue from '@vitejs/plugin-vue';
import { defineConfig } from 'vite';
import dotenv from 'dotenv';

dotenv.config();

const siteAddress = process.env.SITE_ADDRESS;

export default defineConfig({
  plugins: [vue()],
  define: {
    __VUE_OPTIONS_API__: JSON.stringify(true),
    __VUE_PROD_DEVTOOLS__: JSON.stringify(false),
    __VUE_PROD_HYDRATION_MISMATCH_DETAILS__: JSON.stringify(false),
  },
  resolve: {
    alias: {
      vue: 'vue/dist/vue.esm-bundler.js',
    },
  },
  server: {
    port: 3000,
    host: '0.0.0.0',
    hmr: siteAddress
      ? {
          protocol: 'wss',
          host: siteAddress,
          clientPort: 8443,
        }
      : true,
    watch: {
      usePolling: true,
    },
  },
  build: {
    outDir: 'dist',
    emptyOutDir: true,
    rollupOptions: {
      input: getInputs(),
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router'],
        },
      },
    },
  },
});

function getInputs() {
  const inputs = {
    main: resolve(__dirname, 'index.html'),
  };
  if (fs.existsSync(resolve(__dirname, 'signin/index.html'))) {
    inputs.signin = resolve(__dirname, 'signin/index.html');
  }
  if (process.env.NODE_ENV !== 'production' && fs.existsSync(resolve(__dirname, 'dev/index.html'))) {
    inputs.dev = resolve(__dirname, 'dev/index.html');
  }
  return inputs;
}
```

Key notes:
- `@vitejs/plugin-vue` handles SFC compilation (replaces vue-loader + VueLoaderPlugin)
- `resolve.alias` points `vue` to `vue/dist/vue.esm-bundler.js` for runtime template compilation
- `define` sets Vue 3 feature flags for proper tree-shaking
- `server.hmr` with `clientPort` tells the browser which port to connect to without changing where the server listens. Use this when Caddy or another reverse proxy terminates TLS in front of the dev server. Use `port` instead only if you want Vite to bind a separate WebSocket server.
- `server.watch.usePolling: true` for Docker volume mount compatibility (inotify doesn't work across the Docker boundary)
- `manualChunks` splits vendor libs into a separate bundle for caching
- `getInputs()` conditionally includes signin and dev entries
- `public/` directory is served at root automatically (replaces CopyPlugin)
- PostCSS config is auto-loaded by Vite (see below)

## HMR through a reverse proxy

When the dev server runs behind Caddy or nginx with TLS, the browser needs to connect to the proxy's port, not Vite's internal port. Use `clientPort` (not `port`):

```js
// Correct: browser connects to 8443 through Caddy, Vite HMR server stays on 3000
server: {
  hmr: { protocol: 'wss', host: 'myapp.localhost', clientPort: 8443 }
}

// Wrong: Vite tries to create a separate WebSocket server on 8443, which fails inside Docker
server: {
  hmr: { protocol: 'wss', host: 'myapp.localhost', port: 8443 }
}
```

## index.html (project root)

Vite uses the HTML file as the entry point. Add a `<script type="module">` tag pointing to the app entry:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>App</title>
    <link rel="icon" type="image/png" href="/favicon.png" />
  </head>
  <body>
    <div id="app" v-cloak></div>
    <script type="module" src="/js/app.js"></script>
  </body>
</html>
```

For multi-entry projects, each entry gets its own HTML file:

```
ui/
  index.html              # Main app entry
  signin/index.html       # Signin app entry
  dev/index.html          # Dev app entry
```

## postcss.config.js

Vite auto-loads PostCSS config. ESM format (requires `"type": "module"` in `package.json`):

```js
import tailwindcss from 'tailwindcss';
import autoprefixer from 'autoprefixer';
import postcssPresetEnv from 'postcss-preset-env';

export default {
  plugins: [tailwindcss, postcssPresetEnv, autoprefixer],
};
```

Note: Vite requires explicit plugin imports, not string shorthand names.

## tailwind.config.js

ESM format. Content paths include the root `index.html` and all Vue SFCs:

```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './js/**/*.vue'],
  theme: {
    extend: {
      fontFamily: {
        sans: ['"Source Sans 3"', 'Arial', 'sans-serif'],
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        fadeOut: {
          '0%': { opacity: '1' },
          '100%': { opacity: '0' },
        },
        fadeScaleIn: {
          '0%': { opacity: '0', transform: 'scale(0.975)' },
          '100%': { opacity: '1', transform: 'scale(1)' },
        },
      },
      animation: {
        'fade-in': 'fadeIn 0.3s ease',
        'fade-out': 'fadeOut 0.3s ease',
        'fade-scale-in': 'fadeScaleIn 0.3s ease',
      },
      colors: {
        // App-specific colors
      },
    },
  },
  plugins: [],
  safelist: ['hidden', 'opacity-0', 'opacity-100'],
};
```

## CSS Entry (css/style.css)

Tailwind directives and `@layer components` for app-wide component styles:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  input.app-form-input {
    @apply block w-full rounded-md border-0 py-1.5 px-3 bg-white text-gray-900
           shadow-sm ring-1 ring-inset ring-gray-200 placeholder:text-gray-400
           sm:text-sm/6;
  }
  .app-button {
    @apply rounded-md px-3 py-2 bg-gray-100 text-sm font-semibold text-gray-900
           hover:text-red-600 bg-white hover:bg-gray-100 shadow-sm ring-2
           ring-inset ring-gray-300 focus-visible:ring-red-600;
  }
}

[v-cloak] {
  display: none;
}
```

## public/ directory

Static assets served at root. Vite copies these to `dist/` during build:

```
public/
  favicon.png
  icon-192.png
  icon-512.png
  manifest.webmanifest
  sw.js
```

No config needed. Files in `public/` are available at `/` in both dev and production.

## package.json

```json
{
  "type": "module",
  "dependencies": {
    "vue": "^3.5",
    "vue-router": "^4.5",
    "@vitejs/plugin-vue": "^5",
    "vite": "^6",
    "dotenv": "^16",
    "autoprefixer": "^10",
    "postcss": "^8",
    "postcss-preset-env": "^10",
    "prettier": "^3",
    "tailwindcss": "^3"
  },
  "devDependencies": {
    "eslint": "^10",
    "eslint-plugin-vue": "^10"
  }
}
```

`"type": "module"` is required for ESM config files (`vite.config.js`, `postcss.config.js`, `tailwind.config.js`).

## Production build output

Vite produces ESM modules with content hashing:

```
dist/
  index.html                    # With <script type="module"> and <link rel="modulepreload">
  assets/
    vendor-[hash].js             # ESM, vue + vue-router
    index-[hash].js              # ESM, app code
    index-[hash].css             # Extracted CSS
  favicon.png                    # Copied from public/
  ...
```

- `<script type="module">` in production (modern browsers only, no IE11)
- `<link rel="modulepreload">` preloads the vendor chunk, no waterfall
- esbuild minifier by default (much faster than Terser)
- No source maps by default (add `build.sourcemap: true` if needed)

## Troubleshooting

- **Vue warnings about runtime-only build**: Make sure `resolve.alias` points `vue` to `vue/dist/vue.esm-bundler.js`.
- **HMR WebSocket connection failed**: Use `clientPort` (not `port`) when behind a reverse proxy. See HMR section above.
- **PostCSS "Invalid PostCSS Plugin"**: Vite requires explicit plugin imports, not string shorthand names. Use `import tailwindcss from 'tailwindcss'` then `plugins: [tailwindcss]`, not `plugins: ['tailwindcss']`.
- **Tailwind classes not appearing in production**: Add dynamic class names to `safelist` in `tailwind.config.js`.
- **Dev entry in production build**: Conditionally include dev entries in `getInputs()` based on `NODE_ENV`.
- **File watching not working in Docker**: Set `server.watch.usePolling: true`.
