# Vue 2 to Vue 3 Migration

This skill was developed from Vue 2 projects. If you're working on an older project, here's what changes.

## App Creation

```js
// Vue 2
var app = new Vue({
  el: '.app',
  data: { /* ... */ },
});

// Vue 3
var app = Vue.createApp({
  data: function () { return { /* ... */ }; },
});
app.mount('#app');
```

## Router

```js
// Vue 2
var router = new VueRouter({ routes: routes });

// Vue 3
import { createRouter, createWebHistory } from 'vue-router';
var router = createRouter({
  history: createWebHistory(),
  routes: routes,
});
```

## Global Events

Vue 3 removed `$on`, `$off`, and `$emit` from the root instance. Replace with a lib module pattern:

```js
// Instead of vm.$root.$emit('createNotification', params)
// Use:
Notification.create(params);
```

See the notification lib pattern in the main skill document.

## Component Registration

```js
// Vue 2 (CommonJS)
module.exports = {
  components: {
    Btn: require('./btn.vue').default,
  },
};

// Vue 3 (ESM)
import Btn from './btn.vue';

export default {
  components: {
    Btn,
  },
};
```

## Module Format

Migrate from CommonJS to ESM:

```js
// Vue 2
var F = require('../lib/fetch.js');
module.exports = { /* ... */ };

// Vue 3
import F from '../lib/fetch.js';
export default { /* ... */ };
```

## Vue Alias in Webpack

```js
// Vue 2
vue: path.resolve(__dirname, 'node_modules/vue/dist/vue.js')

// Vue 3
vue: path.resolve(__dirname, 'node_modules/vue/dist/vue.esm-bundler.js')
```

## 404 Catch-All Route

```js
// Vue 2
{ path: '*', component: NotFound }

// Vue 3
{ path: '/:pathMatch(.*)*', name: 'Not Found', component: NotFound }
```

## Error Handler

```js
// Vue 2
Vue.config.errorHandler = function (err, vm, info) { /* ... */ };

// Vue 3
app.config.errorHandler = function (err, vm) { /* ... */ };
```

## Keep the Same

These patterns transfer directly -- only the syntax changes:

- Options API component structure (data, methods, computed, watch, mounted, created)
- Loading guards and busy flags
- 3-tier component architecture (layout, page, leaf)
- $root for shared state
- Named routes and programmatic navigation
- Flash messages via navigation state
- lib/ for pure utility functions
- Shell scripts for build workflow
