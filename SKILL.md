---
name: vue-develop
description: Vue SPA development with webpack and Vue 3. Use when building Vue front-end projects, setting up components, routing, state, or build configuration.
---

## Philosophy

- Minimal dependencies. Understand every package in `package.json`.
- Build from the ground up. No CLI scaffolding, no magic generators.
- Simple state over complex state management. Use `$root` until it hurts.
- Shell scripts for build and dev workflow, not npm scripts.
- Pure functions in `lib/` when no reactivity is needed.
- Pragmatic over perfect. Ship working code, iterate later.
- Run webpack builds in Docker. Keep the host clean.

## Tiers

Every choice in this skill falls into one of three tiers. Know which tier you're in.

| Tier | Meaning | Examples |
|------|---------|----------|
| Philosophy | Core to the approach, not negotiable | Minimal deps, 3-tier components, lib/ for pure utils, shell scripts for build, Docker for webpack |
| Pragmatic pattern | Good solution to a real problem, worth repeating | Notification lib, fetch wrapper, contextual redirects, route meta for auth roles |
| Incidental | Could go either way, not part of the philosophy | PascalCase vs camelCase, hash vs history mode, hyphenated filenames |

When in doubt, default to the simplest option and stay consistent within a project.

## Stack

| Purpose | Package | Notes |
|---------|---------|-------|
| Framework | vue | Vue 3, Options API |
| Router | vue-router | Flat routes, named routes, route meta |
| HTTP | fetch | Wrapper in `lib/fetch.js`, not axios |
| Build | webpack 5 | Single config, environment-based |
| CSS | tailwindcss | CSS entry with Tailwind directives, PostCSS pipeline |
| Linting | eslint + prettier | eslint-plugin-vue (flat config), prettier for formatting |

No Vue CLI. No Vuex or Pinia. No axios. No sass.

> **Vue 2 projects**: This skill was developed from Vue 2 projects. Older projects using `new Vue()`, `new VueRouter()`, CommonJS, and `$root.$emit/$on` need migration. See `references/vue2-migration.md`.

## Project Structure

```
ui/
  webpack.config.js          # Single config, dev and production
  postcss.config.js          # PostCSS plugins (tailwindcss, postcss-preset-env)
  tailwind.config.js         # Tailwind content paths, theme, safelist
  eslint.config.js           # ESLint flat config with eslint-plugin-vue
  package.json
  scripts/
    build                    # Shell script: production build
    lint                     # Shell script: eslint + prettier check
    lint-fix                 # Shell script: eslint fix + prettier write
  html/
    index.html               # Main SPA shell
    signin/index.html        # Signin app shell (separate entry)
    dev/index.html           # Dev component app shell (separate entry)
  css/
    style.css                # Entry: @tailwind directives + @layer components
  js/
    app.js                   # Root Vue instance, auth, fetch setup
    router/
      index.js               # Route definitions + beforeEach guard
    lib/                     # Pure utility functions + service modules
    components/              # All .vue SFCs, flat directory
    signin/                  # Separate app entry (signin)
    dev/                     # Separate app entry (component development)
```

## Build and Dev Workflow

Use shell scripts in `scripts/`. Run webpack in Docker.

**scripts/build:**
```sh
#!/bin/sh
NODE_ENV=production npx webpack
```

**scripts/lint:**
```sh
#!/bin/sh
set -e
npx eslint js/ --ext .js,.vue "$@"
npx prettier --check 'js/**/*.{js,vue}' 'css/**/*.css' '*.config.js'
```

**scripts/lint-fix:**
```sh
#!/bin/sh
set -e
npx eslint js/ --ext .js,.vue --fix
npx prettier --write 'js/**/*.{js,vue}' 'css/**/*.css' '*.config.js'
```

For dev server, run `npx webpack serve` or `npm start` (inside Docker).

### Single webpack.config.js

One config file, not two. Use `dotenv` and `process.env.NODE_ENV` to switch between dev and production behavior.

Key patterns:

- **Vendor/app split** via `entry.dependsOn` for cache busting. Vendor bundle changes rarely, app bundle changes often.
- **contenthash filenames** in production, plain names in dev.
- **Vue alias** points to `vue.esm-bundler.js` (full build with template compiler).
- **CSS pipeline**: `postcss-loader` (Tailwind + postcss-preset-env) -> `css-loader` -> `MiniCssExtractPlugin.loader`.
- **HtmlWebpackPlugin** generates HTML with the right chunk imports per entry.
- **devServer config** for local development with `historyApiFallback`.
- **DefinePlugin** for Vue feature flags and environment-specific values.

See `references/webpack-config.md` for the full config.

## Multiple App Entry Points

A project can have separate apps (main, signin, dev) each with their own webpack entry, router, and HTML template. This keeps signin and dev code out of the production bundle.

```js
// webpack.config.js entry
entry: {
  vendor: ['vue', 'vue-router'],
  app: { import: './js/app.js', dependOn: ['vendor'] },
  signin: { import: './js/signin/app.js', dependOn: 'vendor' },
  ...(includeDev ? { dev: { import: './js/dev/app.js' } } : {}),
}
```

Each entry gets its own HtmlWebpackPlugin with `chunks` filtering:

```js
new HtmlWebpackPlugin({
  template: 'html/index.html',
  filename: 'index.html',
  inject: 'body',
  chunks: ['vendor', 'app'],
}),
new HtmlWebpackPlugin({
  template: 'html/signin/index.html',
  filename: 'signin/index.html',
  inject: 'body',
  chunks: ['vendor', 'signin'],
}),
```

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
import LayoutDefault from './layout-default.vue';
import Btn from './btn.vue';

export default {
  components: {
    LayoutDefault,
    Btn,
  },
};
```

## Component Development App

A separate app entry for building components in isolation. Only included in development builds.

The dev app has its own `js/dev/app.js`, its own `html/dev/index.html`, and its own webpack entry. It's completely excluded from the production bundle.

```js
// js/dev/app.js
import '../../css/style.css';
import * as Vue from 'vue';
import DevComponent from '../components/dev-component.vue';
import { createRouter, createWebHashHistory } from 'vue-router';

var routes = [
  { path: '/:component', name: 'Dev Component', component: DevComponent },
];

var router = createRouter({
  history: createWebHashHistory(),
  routes: routes,
});

var app = Vue.createApp({
  template: '<router-view></router-view>',
});

app.use(router);
app.mount('#dev-app');
```

The `dev-component.vue` page renders the named component with mock props using `defineAsyncComponent`:

```vue
<template>
  <div class="p-8 max-w-xl mx-auto">
    <h1 class="text-xl font-bold mb-4">Dev: {{ componentName }}</h1>
    <div v-if="currentComponent" class="mb-6">
      <component
        :is="currentComponent"
        v-bind="mockProps"
        @update:modelValue="onUpdateModelValue"
      />
    </div>
    <div v-else class="text-red-600">
      Unknown component: {{ componentName }}
    </div>
  </div>
</template>

<script>
import { ref, computed, defineAsyncComponent } from 'vue';
import { useRoute } from 'vue-router';

var devComponents = {
  'cb-combobox': defineAsyncComponent(function () {
    return import('./cb-combobox.vue');
  }),
  btn: defineAsyncComponent(function () {
    return import('./btn.vue');
  }),
};

var mockPropsMap = {
  'cb-combobox': {
    id: 'test-combobox',
    slugType: 'customer',
    name: 'Customer',
    modelValue: {},
    items: [],
  },
  btn: {
    variant: 'primary',
    size: 'md',
  },
};

export default {
  setup: function () {
    var route = useRoute();
    var selectedValue = ref({});

    var componentName = computed(function () {
      return route.params.component;
    });

    var currentComponent = computed(function () {
      return devComponents[componentName.value];
    });

    var mockProps = computed(function () {
      var base = mockPropsMap[componentName.value] || {};
      if (base.modelValue !== undefined) {
        return Object.assign({}, base, { modelValue: selectedValue.value });
      }
      return base;
    });

    return {
      componentName: componentName,
      currentComponent: currentComponent,
      mockProps: mockProps,
      selectedValue: selectedValue,
    };
  },
};
</script>
```

Visit `/dev/#/btn` to see the btn component with mock props. Add new components to both `devComponents` and `mockPropsMap` as you build them.

## Routing

### Route Definitions

Flat array, no nesting. Named routes. One file: `js/router/index.js`.

```js
import { createRouter, createWebHistory } from 'vue-router';

import Dashboard from '../components/dashboard.vue';
import Users from '../components/users.vue';
import UsersCreateEdit from '../components/users-create-edit.vue';
import NotFound from '../components/not-found.vue';

const routes = [
  { path: '/', name: 'Dashboard', component: Dashboard },
  { path: '/users', name: 'Users', component: Users },
  { path: '/users/create', name: 'Create User', component: UsersCreateEdit },
  { path: '/users/edit/:email', name: 'Edit User', component: UsersCreateEdit },
  { path: '/:pathMatch(.*)*', name: 'Not Found', component: NotFound },
];

const router = createRouter({
  history: createWebHistory(),
  routes,
});

export default router;
```

### Route Meta for Auth Roles

Use `meta` on routes for auth requirements. A single `beforeEach` guard enforces role checks. This keeps auth logic visible in one place while declaring requirements per-route.

```js
// In route definitions
{
  path: '/admin/employees',
  name: 'Employees',
  component: AdminEmployees,
  meta: {
    requiresAuth: true,
    roles: ['admin'],
  },
}
```

```js
// beforeEach guard
router.beforeEach(function (to, from, next) {
  var requiresAuth = to.meta.requiresAuth;
  var roles = to.meta.roles;

  if (!requiresAuth) {
    return next();
  }

  if (!router.me) {
    return next();
  }

  var userRoles = router.me.roles || [];
  var hasRole = roles.some(function (role) {
    return userRoles.includes(role);
  });

  if (!hasRole) {
    return next({ name: 'Not Found' });
  }

  next();
});
```

Set `router.me` from the root instance after auth completes. The guard runs after the root `created` hook has resolved.

### Contextual Redirect Pattern

When a list view depends on a context (like a selected period or workspace), create a "bare" route that auto-redirects using current app state:

```js
// In route definitions
import ContextualToCurrent from '../components/contextual-to-current.vue';
import Sttpla from '../components/sttpla.vue';

{ name: 'STTPLA Redirect', path: '/sttpla', component: ContextualToCurrent },
{ name: 'STTPLA', path: '/sttpla/:periodId', component: Sttpla },
```

The redirect component reads current context and navigates:

```js
mounted: async function () {
  var vm = this;
  await vm.$root.isReady();
  vm.$router.push(vm.$route.path + '/' + vm.$root.currentPeriod.id);
}
```

### Flash Messages via Navigation State

Pass notification arrays through router navigation. The root `$route` watcher picks them up and creates notifications. This replaces a separate toast or event system for one-shot messages.

**Sending side:**
```js
vm.$router.push({
  name: 'Dashboard',
  params: {
    notifications: [
      { title: 'Saved', message: 'Record updated successfully.' },
    ],
  },
});
```

**Receiving side (root $route watcher):**
```js
watch: {
  $route: function (to) {
    if (to.params.notifications) {
      var vm = this;
      Vue.nextTick(function () {
        to.params.notifications.forEach(function (n) {
          Notification.create(n);
        });
      });
    }
  },
}
```

## State Management

### Default: Root Instance

Put shared state on the root Vue instance. Children access via `this.$root`.

```js
const app = Vue.createApp({
  data: function () {
    return {
      showLoading: false,
      showApp: false,
      me: null,
    };
  },
  methods: {
    hasRole: function (role) {
      return this.me.roles.includes(role);
    },
    toSignin: function () {
      localStorage.setItem('returnTo', '/');
      window.location.href = '/signin';
    },
  },
});
```

Children access:
```js
vm.$root.me;
vm.$root.hasRole('admin');
vm.$root.toSignin();
```

This works well for 3-4 pieces of shared state (auth, context, feature flags). No need for a state management library.

### Cross-Component Communication

- **Parent-child**: props down, events up (`$emit`)
- **Global notifications**: `Notification.create()` via the notification lib module
- **Never**: prop drilling more than 2 levels deep. If you're passing the same prop through 3+ components, put it on root instead.

## API Layer

### Fetch Wrapper

A thin wrapper around native `fetch` in `lib/fetch.js`. Handles auth token injection, JSON body serialization, header defaults, and error normalization.

```js
// lib/fetch.js
let F = {};

F.getIdToken = async function () {
  if (F.idToken) {
    return F.idToken;
  }
  let resp = await globalThis.fetch('/api/authn/session/id_token', {
    method: 'POST',
  }).then(F._assertOk);
  let json = await resp.json();
  F.idToken = json.id_token;
  return F.idToken;
};

F.fetch = async function (url, options) {
  if (!options) options = {};
  if (!options.headers) options.headers = {};
  if (!options.headers['Content-Type']) {
    options.headers['Content-Type'] = 'application/json';
  }

  // Remove undefined headers
  Object.keys(options.headers).forEach(function (key) {
    if (options.headers[key] === undefined) {
      delete options.headers[key];
    }
  });

  // Attach auth token for non-authn routes
  let needsIdToken = !url.startsWith('/api/authn/');
  if (needsIdToken) {
    await F.getIdToken();
    Object.assign(options.headers, {
      Authorization: `Bearer ${F.idToken}`,
    });
  }

  // Serialize body
  if (options.body && typeof options.body !== 'string') {
    options.body = JSON.stringify(options.body);
  }

  let resp = await fetch(url, options).then(F._assertOk);
  return resp;
};

F._assertOk = async function (resp) {
  if (resp.ok) return resp;

  let err = new Error('Unhandled Error');
  let text = await resp.text();
  try {
    let data = JSON.parse(text);
    if (!data.code) throw new Error('not well-formed');
    Object.assign(err, {
      title: data.title,
      message: data.message,
      code: data.code,
      status: resp.status,
      detail: data.detail,
    });
  } catch (_err) {
    Object.assign(err, { detail: text });
  }

  throw err;
};

export default F;
```

Components import and use directly:
```js
import F from '../lib/fetch.js';

// In a method
let resp = await F.fetch('/api/users');
let users = await resp.json();
```

No axios. No interceptors. The wrapper is the interceptor.

See `references/fetch-wrapper.md` for the full pattern with token refresh and 401 retry.

### Domain API Modules

When the app grows, organize API calls into modules under `js/lib/`:

```js
// js/lib/users.js
import F from './fetch.js';

export default {
  list: async function () {
    let resp = await F.fetch('/api/users');
    return resp.json();
  },
  get: async function (email) {
    let resp = await F.fetch('/api/users/' + encodeURIComponent(email));
    return resp.json();
  },
  create: async function (data) {
    return F.fetch('/api/users', { method: 'POST', body: data });
  },
};
```

## Notification System

A lib module that decouples notification creation from the notification component. No `$root.$emit` or `$on` (Vue 3 removed them).

**lib/notification.js:**
```js
let Notification = {
  _component: null,
};

Notification.register = function (component) {
  Notification._component = component;
};

Notification.create = function (params = {}) {
  if (Notification._component) {
    Notification._component.createNotification(params);
  } else {
    console.warn('Notification component not registered.');
  }
};

export default Notification;
```

The Notifications component registers itself on mount:
```js
mounted: function () {
  Notification.register(this);
}
```

Any code can fire a notification:
```js
import Notification from '../lib/notification.js';

Notification.create({
  type: 'fail',
  title: 'Error',
  message: 'Something went wrong.',
});
```

## Utility Functions

`lib/` contains pure functions with no Vue dependency. Date formatting, string helpers, phone number formatting, delay.

```js
// lib/format-date.js
export default function formatDate(dateObj) {
  return dateObj.toLocaleDateString('en-US');
}
```

Import directly into component methods:

```js
import formatDate from '../lib/format-date.js';

export default {
  methods: {
    formatDate,
    // component-specific methods below
    getRecords: async function () { /* ... */ },
  },
};
```

This is simpler than mixins or composables for stateless transformations.

## Error Handling

The root `app.config.errorHandler` catches unhandled errors application-wide:

```js
app.config.errorHandler = async function (err, vm) {
  console.dir(err);

  if (err.status === 401) {
    console.log('unexpected unauthorized response');
    return;
  }

  if (err.message) {
    Notification.create({
      type: err.type || 'fail',
      title: err.title || 'Undocumented Error',
      message: err.message,
    });
    return;
  }

  window.alert(
    'An unexpected error has occurred. Please refresh and try again.'
  );
};
```

Page-level async methods wrap the body in try/catch and surface failures to the user via the notification lib. No bare `await` without error handling. No silent catches.

## Styling

- **Tailwind via PostCSS**. CSS entry contains `@tailwind` directives and `@layer components` for app-wide component styles.
- **Tailwind classes in templates**. No custom CSS unless a component truly needs it.
- **Scoped `<style scoped>`** only when component-specific styles are unavoidable.
- **Safelist** dynamic classes in `tailwind.config.js` if they're constructed at runtime.
- **`v-cloak`** on the mount element. Add `[v-cloak] { display: none; }` in CSS.

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

Every async method wraps the body in try/catch. No bare `await` calls without error handling. No silent catches. Every failure surfaces to the user.

```js
save: async function () {
  let vm = this;
  vm.saveBusy = true;
  try {
    await F.fetch('/api/records/' + vm.record.id, {
      method: 'PATCH',
      body: vm.record,
    });
    vm.$router.push({
      name: 'Records',
      params: {
        notifications: [
          { title: 'Saved', message: 'Record updated successfully.' },
        ],
      },
    });
  } catch (err) {
    Notification.create({
      type: 'fail',
      title: 'Error',
      message: err.message,
    });
  }
  vm.saveBusy = false;
}
```

### Event Handlers

Call `if ($event) $event.preventDefault()` as the first line of event handlers. The `if ($event)` guard makes the method safe to call programmatically (without an event object).

### Lifecycle Hooks

- `created` -- only for bootstrapping that doesn't need the DOM (initial redirects, setting state from route params)
- `mounted` -- data fetching. Always `async`. Data fetches happen after auth is resolved.

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

## Pre-commit

```sh
sh ./scripts/lint
sh ./scripts/build
```

## Guardrails

These are strong defaults, not absolute laws. You can deviate from any of them, but have a reason and discuss it first.

- NEVER add Vuex/Pinia for 2-3 pieces of global state. Use root instance.
- NEVER nest component directories. Flat `components/` folder.
- NEVER use mixins. Use `lib/` for pure functions, root instance for shared state.
- NEVER skip `loading: true` and `v-if="loading"` on page content that depends on fetched data.
- NEVER use `v-if` scattered throughout a template when one loading guard at the top works.
- NEVER make an async API call without try/catch and user-facing error handling.
- ALWAYS use named routes for programmatic navigation.
- ALWAYS include a 404 catch-all route.
- ALWAYS put pure utility functions in `lib/`, not in component methods.
- ALWAYS use shell scripts for build and dev commands.
- ALWAYS use `if ($event) $event.preventDefault()` as first line of event handlers.
- ALWAYS use specific busy flags (`saveBusy`, `deleteBusy`) instead of one generic `busy`.
- ALWAYS run webpack builds in Docker.

## References

- `references/webpack-config.md` -- Full webpack configuration, single config pattern
- `references/page-component-patterns.md` -- Page component templates, create/edit sharing, delete confirmation
- `references/fetch-wrapper.md` -- Fetch wrapper setup, error normalization, token refresh, 401 retry
- `references/vue2-migration.md` -- Notes for migrating Vue 2 projects to Vue 3
