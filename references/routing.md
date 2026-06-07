# Routing Reference

Full routing patterns for Vue Router with auth guards and contextual redirects.

## Full beforeEach Auth Guard

```js
// In js/router/index.js
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

## Contextual Redirect Component

When a list view depends on a context (like a selected period or workspace), create a "bare" route that auto-redirects using current app state.

**Route definitions:**
```js
import ContextualToCurrent from '../components/contextual-to-current.vue';
import Reports from '../components/reports.vue';

{ name: 'Reports Redirect', path: '/reports', component: ContextualToCurrent },
{ name: 'Reports', path: '/reports/:periodId', component: Reports },
```

**Redirect component (`contextual-to-current.vue`):**
```vue
<template>
  <div>Loading...</div>
</template>

<script>
export default {
  mounted: async function () {
    var vm = this;
    await vm.$root.isReady();
    vm.$router.push(vm.$route.path + '/' + vm.$root.currentPeriod.id);
  },
};
</script>
```
