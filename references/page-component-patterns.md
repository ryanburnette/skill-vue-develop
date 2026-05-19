# Page Component Patterns

## Full Page Template

Every page follows this structure:

```js
module.exports = {
  components: {
    LayoutDefault: require('./layout-default.vue').default,
    PageContentHeader: require('./page-content-header.vue').default,
    PageContent: require('./page-content.vue').default,
    Btn: require('./btn.vue').default
  },
  data: function() {
    return {
      loading: true,
      records: [],
      saveBusy: false
    };
  },
  mounted: async function() {
    var vm = this;
    await vm.$root.isReady();
    await vm.getRecords();
    vm.loading = false;
  },
  methods: {
    formatDate,
    shortenStr,
    getRecords: async function() {
      var vm = this;
      try {
        var resp = await vm.$root.axios.get('/api/records');
        vm.records = resp.data;
      } catch (err) {
        vm.$root.unexpectedError(err);
      }
    },
    save: async function(ev) {
      var vm = this;
      if (ev) ev.preventDefault();
      vm.saveBusy = true;
      try {
        await vm.$root.axios.patch('/api/records/' + vm.record.id, vm.record);
        vm.$router.push({
          name: 'Records',
          params: {
            notifications: [
              { title: 'Saved', message: 'Record updated successfully.' }
            ]
          }
        });
      } catch (err) {
        vm.$root.unexpectedError(err);
      }
      vm.saveBusy = false;
    }
  },
  watch: {
    '$root.currentPeriod': async function(to, from) {
      if (!from) return;
      if (to.id !== from.id) {
        vm.$router.push({
          name: 'Records',
          params: { periodId: to.id }
        });
        vm.loading = true;
        await vm.getRecords();
        vm.loading = false;
      }
    }
  }
};
```

## Template Structure

Pages always wrap content in the layout with header and content sections:

```vue
<template>
  <LayoutDefault>
    <PageContentHeader>
      <h1 class="title">Users</h1>
      <p>Description of the page</p>
    </PageContentHeader>
    <PageContent>
      <div v-if="loading">
        <LoadingSpinner />
      </div>
      <div v-else>
        <table>...</table>
      </div>
    </PageContent>
    <PageContent>
      <div class="flex justify-end">
        <Btn variant="primary" :disabled="saveBusy" @click="save">Save</Btn>
      </div>
    </PageContent>
  </LayoutDefault>
</template>
```

## Create/Edit Pages Sharing a Component

List, create, and edit views often share a single component:

```js
// routes.js
{ name: 'Create User', path: '/users/create', component: require('./users-create-edit.vue').default },
{ name: 'Edit User', path: '/users/edit/:email', component: require('./users-create-edit.vue').default }
```

The shared component detects which mode it's in by checking for a route param or comparing paths:

```js
computed: {
  isEdit: function() {
    return this.$route.path.includes('/edit/');
  }
},
mounted: async function() {
  var vm = this;
  await vm.$root.isReady();
  if (vm.isEdit) {
    await vm.getRecord();
  }
  vm.loading = false;
}
```

## Watch for Context Changes

When a page depends on a global context (selected period, workspace, organization), watch `$root` for changes and re-fetch:

```js
watch: {
  '$root.currentPeriod': async function(to, from) {
    if (!from) return;
    if (to.id !== from.id) {
      vm.loading = true;
      await vm.getRecords();
      vm.loading = false;
    }
  }
}
```

This keeps the URL in sync with the selected context and refreshes data automatically.

## Flash Messages After Navigation

After a save or delete, redirect with a notification:

```js
vm.$router.push({
  name: 'Records',
  params: {
    notifications: [
      { title: 'Saved', message: 'Record updated successfully.' }
    ]
  }
});
```

The root `$route` watcher picks these up and emits them to the notification component.

## Delete Confirmation Pattern

```js
deleteRecord: async function(ev) {
  var vm = this;
  if (ev) ev.preventDefault();
  if (!confirm('Are you sure you want to delete this record?')) return;
  vm.deleteBusy = true;
  try {
    await vm.$root.axios.delete('/api/records/' + vm.record.id);
    vm.$router.push({
      name: 'Records',
      params: {
        notifications: [
          { title: 'Deleted', message: 'Record has been deleted.' }
        ]
      }
    });
  } catch (err) {
    vm.$root.unexpectedError(err);
  }
  vm.deleteBusy = false;
}
```