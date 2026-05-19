# API Layer

## Axios Setup

Configure once at the app level with interceptors:

```js
var apiBase;
if (process.env.NODE_ENV === 'production') {
  apiBase = 'https://edvpbs.org';
} else {
  apiBase = 'http://localhost:3080';
}

var axiosInstance = axios.create({
  baseURL: apiBase
});

axiosInstance.interceptors.request.use(function(config) {
  // Attach auth token
  if (app.accessToken) {
    config.headers.Authorization = 'Bearer ' + app.accessToken;
  }
  // Set content type for POST/PATCH
  if (config.method === 'post' || config.method === 'patch') {
    config.headers['Content-Type'] = 'application/json';
  }
  return config;
});
```

Place on root instance:

```js
var app = new Vue({
  el: '.app',
  data: {
    axios: axiosInstance
  }
});
```

Components access via `vm.$root.axios`.

## Standard Error Handler

Every component should use `vm.$root.unexpectedError(err)` in catch blocks. Define it once on the root:

```js
methods: {
  unexpectedError: function(err) {
    console.error(err);
    this.$root.$emit('createNotification', {
      title: 'Error',
      message: 'An unexpected error occurred.'
    });
  }
}
```

## Token Refresh

If using short-lived tokens, handle refresh in the axios interceptor:

```js
axiosInstance.interceptors.request.use(async function(config) {
  // Check if token is stale (e.g., older than 13 minutes)
  if (app.accessTokenCreatedAt) {
    var age = (Date.now() - app.accessTokenCreatedAt) / 1000 / 60;
    if (age > 13) {
      await app.getAccessToken();
    }
  }
  if (app.accessToken) {
    config.headers.Authorization = 'Bearer ' + app.accessToken;
  }
  return config;
});
```

## 401 Handling

Handle unauthorized responses at the component level rather than in a global interceptor. This gives each component control over the user experience:

```js
// In a page component
getRecords: async function() {
  var vm = this;
  try {
    var resp = await vm.$root.axios.get('/api/records');
    vm.records = resp.data;
  } catch (err) {
    if (err.response && err.response.status === 401) {
      vm.$router.push('/login');
      return;
    }
    vm.$root.unexpectedError(err);
  }
}
```

For apps where 401 always means "redirect to login", a response interceptor is simpler:

```js
axiosInstance.interceptors.response.use(
  function(resp) { return resp; },
  function(err) {
    if (err.response && err.response.status === 401) {
      window.location.href = '/login';
    }
    return Promise.reject(err);
  }
);
```

## Domain API Modules

When the app grows, organize API calls into modules:

```
js/api/
  users.js
  records.js
  bidperiods.js
```

Each module:

```js
// js/api/users.js
module.exports = function(axios) {
  return {
    list: function() {
      return axios.get('/api/users');
    },
    get: function(email) {
      return axios.get('/api/users/' + encodeURIComponent(email));
    },
    create: function(data) {
      return axios.post('/api/users', data);
    },
    update: function(email, data) {
      return axios.patch('/api/users/' + encodeURIComponent(email), data);
    }
  };
};
```

Import in the root or page component and pass the axios instance.