# Webpack to Vite Migration

Reference for migrating an existing webpack-based Vue 3 project to Vite. Load this reference when migrating a project.

## Overview

Vite replaces webpack, vue-loader, and a dozen webpack plugins with `@vitejs/plugin-vue` and Vite's built-in capabilities. The migration is mostly mechanical: replace config, move files, update deps.

## File Changes

### Move `html/index.html` to project root

Vite uses the HTML file as the entry point. Add a `<script type="module">` tag:

```html
<!-- Before: html/index.html (HtmlWebpackPlugin template) -->
<div id="app" v-cloak></div>

<!-- After: index.html at project root -->
<div id="app" v-cloak></div>
<script type="module" src="/js/app.js"></script>
```

Delete `html/` directory after migration.

### Move `static/` to `public/`

Vite serves `public/` at root automatically (replaces CopyPlugin):

```
# Before
static/icon-192.png → served at /icon-192.png

# After
public/icon-192.png → served at /icon-192.png
```

### Delete `js/vendor.js`

Vite handles vendor chunking automatically via `build.rollupOptions.output.manualChunks`. The manual vendor entry file is no longer needed.

### Delete `webpack.config.js`

Replaced by `vite.config.js`.

## Config Translation

| webpack | Vite |
|---------|------|
| `VueLoaderPlugin` | `@vitejs/plugin-vue` |
| `HtmlWebpackPlugin` | `<script type="module">` in `index.html` |
| `MiniCssExtractPlugin` | Built-in CSS extraction |
| `CssMinimizerPlugin` + `TerserPlugin` | esbuild minifier (built-in) |
| `CopyPlugin` (static → dist) | `public/` directory |
| `DefinePlugin` | `define` in Vite config |
| `resolve.alias` (vue esm-bundler) | `resolve.alias` in Vite config |
| `devServer` | `server` in Vite config |
| `entry.dependsOn` (vendor split) | `build.rollupOptions.output.manualChunks` |
| `contenthash` filenames | Automatic in production |
| `devtool: 'source-map'` | `build.sourcemap: true` |
| `dotenv.config()` | `dotenv.config()` (same, or use `envDir`) |

## PostCSS Config

Vite auto-loads `postcss.config.js`. Must use explicit plugin imports (not string shorthand):

```js
// Before (webpack CJS)
module.exports = {
  plugins: ['postcss-preset-env', tailwindcss],
};

// After (Vite ESM)
import tailwindcss from 'tailwindcss';
import autoprefixer from 'autoprefixer';
import postcssPresetEnv from 'postcss-preset-env';

export default {
  plugins: [tailwindcss, postcssPresetEnv, autoprefixer],
};
```

`autoprefixer` is typically added during migration (was implicit in some webpack setups).

## Tailwind Config

Update `content` paths (no more `html/` directory):

```js
// Before
content: ['./html/**/*.html', './js/**/*.vue'],

// After
content: ['./index.html', './js/**/*.vue'],
```

Convert to ESM if using `"type": "module"` in package.json.

## HMR Config

The most common migration pitfall. When the dev server runs behind Caddy or nginx with TLS:

```js
// WRONG: port makes Vite try to create a separate WS server on 8443
server: {
  hmr: { protocol: 'wss', host: 'mysite.localhost', port: 8443 }
}

// CORRECT: clientPort tells the browser where to connect, WS stays on 3000
server: {
  hmr: { protocol: 'wss', host: 'mysite.localhost', clientPort: 8443 }
}
```

Also set `server.watch.usePolling: true` for Docker volume compatibility (replaces `WATCHPACK_POLLING=true` env var).

## Caddy Config

Remove any `/ws` handler for the webpack dev server WebSocket. Vite's HMR passes through the main proxy:

```Caddyfile
# Before (webpack)
handle /ws {
    reverse_proxy localhost:3000
}
handle {
    reverse_proxy localhost:3000
}

# After (Vite)
handle {
    reverse_proxy localhost:3000
}
```

## package.json Changes

Add `"type": "module"` for ESM configs. Swap webpack deps for Vite:

```json
{
  "type": "module",
  "dependencies": {
    "vue": "^3.5",
    "vue-router": "^4.5",
    "@vitejs/plugin-vue": "^5",
    "vite": "^6",
    "autoprefixer": "^10",
    "postcss": "^8",
    "postcss-preset-env": "^10",
    "prettier": "^3",
    "tailwindcss": "^3"
  }
}
```

Removed: `@vue/compiler-sfc`, `copy-webpack-plugin`, `css-loader`, `css-minimizer-webpack-plugin`, `html-webpack-plugin`, `mini-css-extract-plugin`, `postcss-loader`, `terser-webpack-plugin`, `vue-loader`, `webpack`, `webpack-cli`, `webpack-dev-server`.

## Scripts

```sh
# scripts/build
# Before
NODE_ENV=production npx webpack

# After
NODE_ENV=production npx vite build

# dev server
# Before
npx webpack serve

# After
npx vite
```

## Production Build Differences

| Aspect | webpack | Vite |
|--------|---------|------|
| Module format | IIFE (`<script>`) | ESM (`<script type="module">`) |
| Minifier | Terser | esbuild (faster) |
| Source maps | On by default | Off by default |
| Runtime chunk | Separate file | Merged into vendor |
| `keep_fnames`/`keep_classnames` | Via Terser config | Via `build.esbuild.keepNames: true` |
| Asset hashing | `[contenthash]` in config | Automatic |

ESM modules require modern browsers (no IE11). Source maps can be enabled with `build.sourcemap: true` if needed.

## Migration Checklist

1. Add `vite` and `@vitejs/plugin-vue` to `package.json`, add `"type": "module"`
2. Create `vite.config.js` (translate from `webpack.config.js`)
3. Move `html/index.html` to project root, add `<script type="module">`
4. Move `static/` to `public/`
5. Delete `js/vendor.js`
6. Update `postcss.config.js` to ESM with explicit imports
7. Update `tailwind.config.js` content paths and convert to ESM
8. Update `scripts/build` and dev server command
9. Remove webpack deps from `package.json`
10. Remove `/ws` handler from Caddy dev config
11. Rebuild Docker image (`npm install` with new deps)
12. Test dev server with HMR through proxy
13. Test production build
14. Deploy and verify
