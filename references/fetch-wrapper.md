# Fetch Wrapper

A thin wrapper around native `fetch` that handles auth token injection, JSON serialization, and error normalization. No axios.

## Basic Pattern

```js
// lib/fetch.js
'use strict';

let F = {};

F.getIdToken = async function () {
  if (F.idToken) {
    return F.idToken;
  }

  let resp = await globalThis
    .fetch('/api/authn/session/id_token', {
      method: 'POST',
    })
    .then(F._assertOk);

  let json = await resp.json();
  F.idToken = json.id_token;
  return F.idToken;
};

F.fetch = async function (url, options) {
  if (!options) {
    options = {};
  }

  if (!options.headers) {
    options.headers = {};
  }
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
  if (resp.ok) {
    return resp;
  }

  let err = new Error('Unhandled Error');
  let text = await resp.text();
  try {
    let data = JSON.parse(text);
    if (!data.code) {
      throw new Error('not well-formed');
    }
    Object.assign(err, {
      title: data.title,
      message: data.message,
      code: data.code,
      status: resp.status,
      detail: data.detail,
    });
  } catch (_err) {
    Object.assign(err, {
      detail: text,
    });
  }

  throw err;
};

export default F;
```

## Token Refresh and 401 Retry

For apps that need token refresh, extend the wrapper:

```js
// lib/fetch-utils.js
'use strict';

let idToken = null;

export async function getIdToken() {
  if (idToken) {
    return idToken;
  }

  const resp = await apiFetch('/api/authn/session/id_token', {
    method: 'POST',
  });

  if (resp.ok) {
    let data = await resp.json();
    idToken = data.id_token;
    return idToken;
  }

  if (resp.code === 'E_SESSION_INVALID') {
    return null;
  }
}

export function clearIdToken() {
  idToken = null;
}

export async function apiFetch(url, options) {
  if (!options) options = {};
  if (!options.method) options.method = 'GET';
  if (!options.headers) options.headers = {};
  if (!options.headers['Content-Type']) {
    options.headers['Content-Type'] = 'application/json';
  }

  Object.keys(options.headers).forEach(function (key) {
    if (options.headers[key] === undefined) {
      delete options.headers[key];
    }
  });

  const resp = await fetch(url, options);

  if (resp.ok) {
    return resp;
  }

  const err = new Error('Unhandled Error');
  const text = await resp.text();

  try {
    const data = JSON.parse(text);
    if (!data.code) throw new Error('Invalid JSON response');
    Object.assign(err, {
      message: data.message,
      code: data.code,
      status: resp.status,
      detail: data.detail,
    });
  } catch (_err) {
    Object.assign(err, {
      detail: text || 'Unexpected error with no additional details.',
    });
  }

  if (err.status < 500) {
    return err;
  }

  throw err;
}

export async function apiAuthFetch(url, options) {
  const token = await getIdToken();

  if (!options) options = {};
  if (!options.headers) options.headers = {};
  options.headers.Authorization = 'Bearer ' + token;

  try {
    return await apiFetch(url, options);
  } catch (err) {
    if (err.status === 401) {
      clearIdToken();
      const newToken = await getIdToken();
      options.headers.Authorization = 'Bearer ' + newToken;
      return await apiFetch(url, options);
    }
    throw err;
  }
}
```

## Error Handling Convention

The fetch wrapper throws errors with structured properties when the API returns a well-formed error response. The consuming code can check `err.code` for specific error handling:

```js
try {
  let resp = await F.fetch('/api/records', { method: 'POST', body: data });
  // success
} catch (err) {
  if (err.code === 'E_DUPLICATE') {
    Notification.create({
      type: 'fail',
      title: 'Duplicate',
      message: err.message,
    });
    return;
  }
  Notification.create({
    type: 'fail',
    title: 'Error',
    message: err.message,
  });
}
```

For non-OK responses under status 500, the `fetch-utils` pattern returns the error object instead of throwing. The consuming code checks `resp.ok`:

```js
let resp = await apiFetch('/api/records');
if (!resp.ok) {
  // resp is the error object with code, message, status, detail
  Notification.create({ type: 'fail', message: resp.message });
  return;
}
let data = await resp.json();
```

Choose one convention per project and stick with it. The `F.fetch` pattern (always throw on non-OK) is simpler for most pages. The `apiFetch` pattern (return errors under 500) gives more control for flows like signin where you want to handle specific codes without try/catch everywhere.
