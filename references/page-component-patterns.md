# Page Component Patterns

## Full Page Template

Every page follows this structure:

```vue
<template>
  <LayoutDefault>
    <PageContentHeader>
      <h1 class="text-xl font-medium text-gray-900">Users</h1>
      <p class="mt-1 text-sm text-gray-500">Manage user accounts</p>
    </PageContentHeader>
    <PageContent>
      <div v-if="loading">
        <LoadingSpinner />
      </div>
      <div v-else>
        <!-- table, cards, etc -->
      </div>
    </PageContent>
    <PageContent>
      <div class="flex justify-end">
        <Btn variant="primary" :disabled="saveBusy" @click="save">Save</Btn>
      </div>
    </PageContent>
  </LayoutDefault>
</template>

<script>
import F from '../lib/fetch.js';
import Notification from '../lib/notification.js';
import LayoutDefault from './layout-default.vue';
import PageContentHeader from './page-content-header.vue';
import PageContent from './page-content.vue';
import Btn from './btn.vue';
import LoadingSpinner from './loading-spinner.vue';
import formatDate from '../lib/format-date.js';

export default {
  components: {
    LayoutDefault,
    PageContentHeader,
    PageContent,
    Btn,
    LoadingSpinner,
  },
  data: function () {
    return {
      loading: true,
      records: [],
      saveBusy: false,
    };
  },
  mounted: async function () {
    let vm = this;
    await vm.getRecords();
    vm.loading = false;
  },
  methods: {
    formatDate,
    getRecords: async function () {
      let vm = this;
      try {
        let resp = await F.fetch('/api/records');
        vm.records = await resp.json();
      } catch (err) {
        Notification.create({
          type: 'fail',
          title: 'Error',
          message: err.message,
        });
      }
    },
    save: async function ($event) {
      let vm = this;
      if ($event) $event.preventDefault();
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
    },
  },
  watch: {
    '$root.currentPeriod': async function (to, from) {
      let vm = this;
      if (!from) return;
      if (to.id !== from.id) {
        vm.$router.push({
          name: 'Records',
          params: { periodId: to.id },
        });
        vm.loading = true;
        await vm.getRecords();
        vm.loading = false;
      }
    },
  },
};
</script>
```

## Create/Edit Pages Sharing a Component

List, create, and edit views often share a single component:

```js
// router/index.js
{ path: '/users/create', name: 'Create User', component: UsersCreateEdit },
{ path: '/users/edit/:email', name: 'Edit User', component: UsersCreateEdit },
```

The shared component detects which mode it's in by checking for a route param:

```js
computed: {
  isEdit: function () {
    return this.$route.path.includes('/edit/');
  },
},
mounted: async function () {
  let vm = this;
  if (vm.isEdit) {
    await vm.getRecord();
  }
  vm.loading = false;
},
```

## Watch for Context Changes

When a page depends on a global context (selected period, workspace, organization), watch `$root` for changes and re-fetch:

```js
watch: {
  '$root.currentPeriod': async function (to, from) {
    let vm = this;
    if (!from) return;
    if (to.id !== from.id) {
      vm.loading = true;
      await vm.getRecords();
      vm.loading = false;
    }
  },
},
```

This keeps the URL in sync with the selected context and refreshes data automatically.

## Flash Messages After Navigation

After a save or delete, redirect with a notification:

```js
vm.$router.push({
  name: 'Records',
  params: {
    notifications: [
      { title: 'Saved', message: 'Record updated successfully.' },
    ],
  },
});
```

The root `$route` watcher picks these up and creates notifications via the notification lib.

## Delete Confirmation Pattern

```js
deleteRecord: async function ($event) {
  let vm = this;
  if ($event) $event.preventDefault();
  if (!confirm('Are you sure you want to delete this record?')) return;
  vm.deleteBusy = true;
  try {
    await F.fetch('/api/records/' + vm.record.id, {
      method: 'DELETE',
    });
    vm.$router.push({
      name: 'Records',
      params: {
        notifications: [
          { title: 'Deleted', message: 'Record has been deleted.' },
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
  vm.deleteBusy = false;
},
```

## Accessing Root State

Children access root state and methods via `this.$root`:

```js
computed: {
  isAdmin: function () {
    return this.$root.hasRole('admin');
  },
  me: function () {
    return this.$root.me;
  },
},
```
