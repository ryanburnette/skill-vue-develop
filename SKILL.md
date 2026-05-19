---
name: vue-develop
description: Vue SPA development with webpack, Vue 2/3, and Vue Router. Use when building Vue front-end projects, setting up components, routing, state, or build configuration.
---

## Philosophy

- Minimal dependencies. Understand every package in `package.json`.
- Build from the ground up. No CLI scaffolding, no magic generators.
- Simple state over complex state management. Reach for the simplest tool.
- Shell scripts for build and dev workflow, not npm scripts.
- Pure functions in `lib/` when no reactivity is needed.
- Pragmatic over perfect. Ship working code, iterate later.

## Tiers

Every choice in this skill falls into one of three tiers. Know which tier you're in.

| Tier | Meaning | Examples |
|------|---------|----------|
| Philosophy | Core to the approach, not negotiable | Minimal deps, 3-tier components, lib/ for pure utils, shell scripts for build |
| Pragmatic pattern | Good solution to a real problem, worth repeating | Flash messages via navigation state, contextual redirects, auth at app level |
| Incidental | Could go either way, not part of the philosophy | PascalCase vs camelCase, hash vs history mode, hyphenated filenames, CommonJS vs ESM |

When in doubt, default to the simplest option and stay consistent within a project.

## Stack

| Purpose | Package | Notes |
|---------|---------|-------|
| Framework | vue | Vue 2 or 3, Options API or Composition API |
| Router | vue-router | Flat routes, named routes |
| HTTP | axios | Configured at app level with interceptors |
| Build | webpack + webpack-merge | Separate dev and production configs |
| CSS | tailwindcss + sass | SCSS entry with Tailwind directives, PostCSS pipeline |
| Linting | prettier | No ESLint, format only |
| Asset hashing | @ryanburnette/hash-assets | Post-build step, not webpack contenthash |

No Vue CLI. No Vuex or Pinia unless the app genuinely needs them (5+ pieces of shared state with complex interactions).

## Project Structure

```
ui/
  webpack.config.js          # Dev/base config
  webpack.production.js      # Production override (merged)
  postcss.config.js          # PostCSS plugins (tailwindcss, postcss-preset-env)
  tailwind.config.js         # Tailwind content paths, theme, safelist
  package.json
  scripts/
    development              # Shell script: build + watch
    build                    # Shell script: production build
  html/
    index.html               # SPA shell
  css/
    style.scss               # Entry: @tailwind directives only
  js/
    app.js                   # Root Vue instance, auth, axios setup
    routes.js                # Route definitions
    api/                     # API modules per domain (optional)
    lib/                     # Pure utility functions (no Vue dependency)
    components/              # All .vue SFCs, flat directory
```

## Build and Dev Workflow

Use shell scripts in `scripts/`, not npm scripts.

**scripts/development:**
```sh
#!/bin/bash
rm -rf dist
mkdir dist
cp html/index.html dist/
npx webpack --watch
```

**scripts/build:**
```sh
#!/bin/bash
rm -rf dist
mkdir dist
cp html/index.html dist/
NODE_ENV=production npx webpack --config webpack.production.js
npx hash-assets dist/
```

### webpack.config.js (base/dev)

Key patterns:

- **Vendor/app split** via `entry.dependsOn` for cache busting. Vendor bundle changes rarely, app bundle changes often.
- **Vue alias** points to full build (with template compiler) for dev, minified for production.
- **SCSS pipeline**: sass-loader -> postcss-loader (Tailwind + postcss-preset-env) -> css-loader -> MiniCssExtractPlugin.loader
- **No HtmlWebpackPlugin**. HTML is copied manually and imported as an asset/resource module.
- **No devServer config**. Watch mode + separate web server (backend or Caddy).

See `references/webpack-config.md` for the full config.

### webpack.production.js (override)

Uses `webpack-merge` on top of base config:

```js
var merge = require('webpack-merge');
var config = merge(require('./webpack.config.js'), {
  mode: 'production',
  resolve: {
    alias: {
      vue: path.resolve(__dirname, 'node_modules/vue/dist/vue.min.js')
    }
  },
  optimization: {
    minimize: true,
    minimizer: [new CssMinimizerPlugin(), new TerserPlugin()]
  }
});
module.exports = config;
```

### Environment-based Backend URL

When the front end is separate from the back end, configure the API base URL per environment:

```js
// In app.js or a dedicated config module
var apiBase;
if (process.env.NODE_ENV === 'production') {
  apiBase = 'https://edvpbs.org';
} else {
  apiBase = 'http://localhost:3080';
}
```

Axios uses this as the base URL. All API calls become relative: `axios.get('/api/users')` resolves against the configured base.

## Component Architecture

Three tiers. Every component is one of these.

### Layout

One layout component per app. Wraps `<slot>` for page content. Contains sidebar, top bar, notifications -- everything that persists across routes.

```vue
<template>
  <div class="flex">
    <NavX />
    <main class="flex-1">
      <slot></slot>
    </main>
    <Notifications />
  </div>
</template>
```

Pages always wrap content in the layout:

```vue
<template>
  <LayoutDefault>
    <PageContentHeader>
      <h1 class="title">Users</h1>
    </PageContentHeader>
    <PageContent>
      <!-- table, cards, etc -->
    </PageContent>
  </LayoutDefault>
</template>
```

### Page

Route targets. Each route maps to one page component. Pages fetch their own data in `mounted`, compose leaf components, and wrap everything in the layout.

### Leaf

Reusable primitives. No route awareness. Receive data via props, communicate up via events.

- UI primitives: `btn`, `modal`, `form-control`, `input-text`, `input-checkbox`
- Display helpers: `boolean-icon`, `loading-spinner`, `heroicon`
- Domain-specific reusable: `positions-table`, `bidperiods-dropdown`

### Registration

Import locally in the component that uses them. No global registration.

```js
components: {
  LayoutDefault: require('./layout-default.vue').default,
  Btn: require('./btn.vue').default
}
```

## Component Development Route

A dev-only route for building components in isolation. Only registered in development builds.

```js
// In routes.js
var devRoutes = [];
if (process.env.NODE_ENV !== 'production') {
  devRoutes = [
    {
      name: 'Dev Component',
      path: '/dev/:component',
      component: require('./components/dev-component.vue').default
    }
  ];
}
// ... spread devRoutes into the routes array
```

The `dev-component.vue` page renders the named component with mock props:

```vue
<template>
  <div>
    <component v-bind:is="currentComponent" v-bind="mockProps"></component>
  </div>
</template>

<script>
var mockProps = require('./dev-props.js');
var components = {
  Btn: require('./btn.vue').default,
  Modal: require('./modal.vue').default,
  // Add new components here as you build them
};

module.exports = {
  computed: {
    currentComponent: function() {
      return components[this.$route.params.component];
    },
    mockProps: function() {
      return mockProps[this.$route.params.component] || {};
    }
  }
};
</script>
```

Mock props live in `dev-props.js`:

```js
module.exports = {
  Btn: { variant: 'primary', size: 'md' },
  Modal: { visible: true, title: 'Test Modal' }
};
```

Visit `/dev/Btn` to see the btn component with mock props. Add new components to both the `components` map and `dev-props.js` as you build them.

## Routing

### Route Definitions

Flat array, no nesting. Named routes. One file: `routes.js`.

```js
module.exports = [
  { name: 'Dashboard', path: '/', component: require('./components/dashboard.vue').default },
  { name: 'Users', path: '/users', component: require('./components/users.vue').default },
  { name: 'Create User', path: '/users/create', component: require('./components/users-create-edit.vue').default },
  { name: 'Edit User', path: '/users/edit/:email', component: require('./components/users-create-edit.vue').default },
  { name: 'Not Found', path: '*', component: require('./components/not-found.vue').default }
];
```

### Contextual Redirect Pattern

When a list view depends on a context (like a selected period or workspace), create a "bare" route that auto-redirects using current app state:

```js
// routes.js -- the bare path points to a redirect component
{ name: 'STTPLA Redirect', path: '/sttpla', component: require('./components/contextual-to-current.vue').default },
{ name: 'STTPLA', path: '/sttpla/:periodId', component: require('./components/sttpla.vue').default }
```

The redirect component reads current context and navigates:

```js
// contextual-to-current.vue
mounted: async function() {
  var vm = this;
  await vm.$root.isReady();
  vm.$router.push(vm.$route.path + '/' + vm.$root.currentPeriod.id);
}
```

### Flash Messages via Navigation State

Pass notification arrays through router navigation, consumed at the app level. This replaces the need for a separate toast or event system for one-shot messages.

**Sending side:**
```js
vm.$router.push({
  name: 'Dashboard',
  params: {
    notifications: [
      { title: 'Saved', message: 'Record updated successfully.' }
    ]
  }
});
```

**Receiving side (root $route watcher):**
```js
watch: {
  $route: function(to) {
    if (to.params.notifications) {
      var vm = this;
      Vue.nextTick(function() {
        to.params.notifications.forEach(function(n) {
          vm.$root.$emit('createNotification', n);
        });
      });
    }
  }
}
```

### Auth at App Level, Not Router Guards

No `beforeEach` or `beforeEnter` guards. Auth logic lives in the root instance:

- Root `created` hook checks auth state and redirects to login if needed
- Root `$route` watcher handles redirects on navigation
- Components handle 401 responses individually
- Axios interceptors handle token refresh and unauthorized responses

This keeps auth logic in one visible place instead of scattered across route definitions and guard functions.

## State Management

### Default: Root Instance

Put shared state on the root Vue instance. Children access via `this.$root`.

```js
var app = new Vue({
  el: '.app',
  data: {
    ready: false,
    user: null,
    axios: axiosInstance
  },
  methods: {
    isReady: async function() { /* ... */ },
    unexpectedError: function(err) { /* ... */ }
  }
});
```

This works well for 3-4 pieces of shared state (auth, context, axios). Children access:

```js
await vm.$root.isReady();
vm.$root.axios.get('/api/users');
vm.$root.unexpectedError(err);
```

### When to Use Pinia

Reach for Pinia when:
- 5+ pieces of shared state with complex interactions
- Multiple independent stores make more sense than one root
- Devtools inspection would help debugging

Don't use Pinia just because "that's how Vue apps are supposed to work."

### Cross-Component Communication

- **Parent-child**: props down, events up (`$emit`)
- **Global events**: `$root.$emit` / `$root.$on` for notifications (or a tiny emitter library for Vue 3, which removed these methods)
- **Never**: prop drilling more than 2 levels deep. If you're passing the same prop through 3+ components, put it on root/state instead.

## API Layer

Configure axios once at the app level with interceptors. Place it on the root instance: `data: { axios: axiosInstance }`. Components call: `vm.$root.axios.get('/api/users')`.

For domain-specific API modules when the app grows, create `js/api/users.js` etc., each exporting functions that use the shared axios instance.

See `references/api-layer.md` for the full axios setup pattern.

## Utility Functions

`lib/` contains pure functions with no Vue dependency. Date formatting, string helpers, file size display.

```js
// lib/format-date.js
module.exports = function formatDate(dateObj) {
  return dateObj.toLocaleDateString('en-US');
};
```

Import directly into component methods:

```js
methods: {
  formatDate,
  shortenStr,
  // component-specific methods below
  getRecords: async function() { /* ... */ }
}
```

This is simpler than mixins or composables for stateless transformations.

## Styling

- **Tailwind via PostCSS**. SCSS entry contains only `@tailwind base/components/utilities`.
- **Tailwind classes in templates**. No custom CSS unless a component truly needs it.
- **Scoped `<style scoped>`** only when component-specific styles are unavoidable.
- **Safelist** dynamic classes in `tailwind.config.js` if they're constructed at runtime.

## Writing Vue Components

These are the patterns that define how components are written. They're not incidental style -- they're deliberate choices that make code predictable and debuggable.

### Loading Guards

Every page starts with `loading: true` and wraps its content in `v-if="loading"` / `v-else`. One guard at the top of the template, not `v-if="record"` scattered throughout. This prevents rendering data that hasn't loaded yet.

### Busy Flags for Actions

Distinguish between initial load and action in progress. Use specific names:
- `loading: true` -- initial page load, starts true
- `saveBusy: false` -- form submission in progress
- `deleteBusy: false` -- delete action in progress
- `uploadBusy: false` -- file upload in progress

Disable the corresponding button while busy. Set to true at the start, false after the action completes.

### Async Methods

Every async method wraps the body in try/catch and calls `vm.$root.unexpectedError(err)`. No bare `await` calls without error handling. No silent catches. Every failure surfaces to the user.

### Event Handlers

Call `if (ev) ev.preventDefault()` as the first line of event handlers. The `if (ev)` guard makes the method safe to call programmatically (without an event object).

### Lifecycle Hooks

- `created` -- only for bootstrapping that doesn't need the DOM (initial redirects, setting state from route params)
- `mounted` -- data fetching. Always `async`. Always `await vm.$root.isReady()` before any API calls.

### Method Naming

- Data-fetching methods: verb + noun (`getRecords`, `getUser`, `getBidPeriods`)
- Action methods: verb only when context is clear (`save`, `delete`), verb + noun otherwise (`deleteRecord`, `uploadFile`)
- Utility imports: noun or verb directly (`formatDate`, `shortenStr`)

### Utility Functions in Methods

Import pure functions from `lib/` directly into the `methods` object. This makes them available in templates as `{{ formatDate(record.date) }}` without polluting `data` or `computed`.

### Props

Always use object syntax with type and default. Array and object defaults use the function form to avoid shared references between component instances.

### Watch for Context Changes

When a page depends on global context (selected period, workspace), watch `$root` and re-fetch. Guard with `if (!from) return` to skip initial trigger, and `if (to.id !== from.id)` to prevent no-op re-fetches.

See `references/page-component-patterns.md` for full code examples of all these patterns.

## Page Component Pattern

- Always wrap content in `<LayoutDefault>` with `<PageContentHeader>` and `<PageContent>`
- See `references/page-component-patterns.md` for the full template

## Pre-commit

```sh
npx webpack --config webpack.production.js
npx prettier --check 'js/**/*.{js,vue}'
```

## Guardrails

These are strong defaults, not absolute laws. You can deviate from any of them, but have a reason and discuss it first.

- NEVER add Vuex/Pinia for 2-3 pieces of global state. Use root instance.
- NEVER use route guards for auth. Handle at app level.
- NEVER nest components directories. Flat `components/` folder.
- NEVER use mixins. Use `lib/` for pure functions, root instance for shared state.
- NEVER skip `await vm.$root.isReady()` before API calls in page components.
- NEVER make an async API call without try/catch and `unexpectedError`.
- NEVER skip `v-if="loading"` on page content that depends on fetched data.
- NEVER use `v-if` scattered throughout a template when one loading guard at the top works.
- ALWAYS use named routes for programmatic navigation.
- ALWAYS include a 404 catch-all route.
- ALWAYS put pure utility functions in `lib/`, not in component methods.
- ALWAYS use shell scripts for build and dev commands.
- ALWAYS use `if (ev) ev.preventDefault()` as first line of event handlers.
- ALWAYS use specific busy flags (`saveBusy`, `deleteBusy`) instead of one generic `busy`.

## Future Considerations

These are migration paths for when the project is ready to evolve. Do not implement until the team is ready.

### Vue 2 to Vue 3

- `$root` state pattern -> `provide`/`inject` or Pinia
- `$root.$emit/$on` -> mitt or tiny emitter library (Vue 3 removed these)
- `new Vue()` -> `createApp()`
- `new VueRouter()` -> `createRouter()`
- Options API -> Composition API with `<script setup>` (incremental, component by component)
- The writing conventions in this skill are Options API. Composition API replaces `var vm = this` with reactive refs, `module.exports` with `export default`, and `data: function()` with `ref()`/`reactive()`. The patterns (loading guards, busy flags, async try/catch discipline, event handler conventions) transfer directly -- only the syntax changes.

### Webpack to Vite

Vite is less complex than webpack. The same philosophy of "understand your tools" is better served by Vite because there's less to understand.

- Config drops from ~60 lines to ~15
- No babel config, no loader rules, no merge strategy
- HMR and source maps work out of the box
- `.env.*` files built in for per-environment config (`VITE_API_URL`)
- Production build is Rollup-based, fast
- Post-build hash-assets step still works, or use Vite's built-in content hashing

### Improvements from the Vue 2/webpack Era

These were issues in the reference project that should be addressed in new projects:

- Add source maps (`devtool: 'source-map'` in webpack, or Vite default)
- Add a 404 catch-all route
- Use `meta` fields on routes for page titles and auth hints
- Use history mode with server-side fallback (instead of hash mode)
- Add lazy loading with `() => import()` for route components in larger apps
- Use consistent import style (all ESM or all CJS, not mixed)
- Remove dead loader rules that don't match any files

## References

- `references/webpack-config.md` -- Full webpack configuration patterns for dev and production
- `references/page-component-patterns.md` -- Page component templates, create/edit sharing, delete confirmation
- `references/api-layer.md` -- Axios setup, interceptors, token refresh, domain API modules