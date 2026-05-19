# Notification System Reference

A lib module that decouples notification creation from the notification component. No `$root.$emit` or `$on` (Vue 3 removed them).

## lib/notification.js

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

## Notifications Component Registration

The Notifications component registers itself on mount:

```js
mounted: function () {
  Notification.register(this);
}
```

## Usage

Any code can fire a notification:

```js
import Notification from '../lib/notification.js';

Notification.create({
  type: 'fail',
  title: 'Error',
  message: 'Something went wrong.',
});
```
