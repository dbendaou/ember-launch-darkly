# ember-launch-darkly

**NOTE: This repo is very much taking a "README Driven Development" approach. This addon will exist soon, but currently we are simply understanding how we want it to work.**

Feel free to PR this README with any suggestions to how you think we can improve the implementation.

<hr>


This addon wraps the [Launch Darkly](https://launchdarkly.com/) feature flagging service and provides helpers to implement feature flagging in your application

## Installation

```bash
$ ember install ember-launch-darkly
```

## Configuration

ember-launch-darkly can be configured from `config/environment.js` as follows:

```js
module.exports = function(environment) {
  let ENV = {
    launchDarkly: {
      // options
    }
  };

  return ENV
};
```

ember-launch-darkly supports the following configuration options:

### `clientSideId` (required)

The client side ID generated by Launch Darkly which is available in your [account settings page](https://app.launchdarkly.com/settings#/projects). See the Launch Darkly docs for [more information on how the client side ID is used](https://docs.launchdarkly.com/docs/js-sdk-reference#section-initializing-the-client).

### `local`

Specify that you'd like to pull feature flags from your local config instead of remotely from Launch Darkly. This is likely appropriate when running in the `development` environment or an external environment for which you don't have Launch Darkly set up.

This option will also make the launch darkly service available in the browser console so that feature flags can be enabled/disabled manually.

_Default_: `false` in production, `true` in all other environments

### `localFeatureFlags`

A list of initial values for your feature flags. This property is only used when `local: true` to populate the list of feature flags for environments such as local development where it's not desired to store the flags in Launch Darkly.

_Default_: `null`

### `secureMode`

Enable secure mode to ensure that feature flag settings for a user are kept private. See the Launch Darkly docs for [more information on secure mode](https://docs.launchdarkly.com/docs/js-sdk-reference#section-secure-mode).

## Usage

### Initialization

Before being used, Launch Darkly must be initialized. This should happen early so choose an appropriate place to make the call such as an application initializer or the application route.

The `initialize()` function returns a promise that resolves when the Launch Darkly client is ready so Ember will wait until this happens before proceeding.

The user `key` is the only required attribute, see the [Launch Darkly documentation](https://docs.launchdarkly.com/docs/js-sdk-reference#section-users) for the other attributes you can provide.

```js
// /app/application/route.js

import Route from 'ember-route';
import service from 'ember-service/inject';

export default Route.extend({
  launchDarkly: service(),

  model() {
    let user = {
      key: 'aa0ceb',
      anonymous: true
    };

    return this.get('launchDarkly').initialize(user);
  }
});
```

### Identify

If you initialized Launch Darkly with an anonymous user and want to re-initialize it for a specific user to receive the flags for that user, you can use the `identify`. This can only be called after `initialization` has been called.

```js
// /app/session/route.js

import Route from 'ember-route';
import service from 'ember-service/inject';

export default Route.extend({
  session: service(),
  launchDarkly: service(),

  model() {
    return this.get('session').getSession();
  },

  afterModel(session) {
    let user = {
      key: session.get('user.id'),
      firstName: session.get('user.firstName'),
      email: session.get('user.email')
    };

    return this.get('launchDarkly').identify(user);
  }
});
```

### Templates

ember-launch-darkly provides a `variation` template helper to check you feature flags.

If your feature flag is a boolean based flag, you might use it in an `{{if}}` like so:

```hbs
{{#if (variation "new-login-screen")}}
  {{login-screen}}
{{else}}
  {{old-login-screen}}
{{/if}}
```

If your feature flag is a multivariate based flag, you might use it in an `{{with}}` like so:

```hbs
{{#with (variation "new-login-screen") as |variant|}}
  {{#if (eq variant "login-screen-a")}
    {{login-screen-a}}
  {{else if (eq variant "login-screen-b")}}
    {{login-screen-b}}
  {{/if}}
{{else}}
  {{login-screen}}
{{/with}}
```

### Javascript

ember-launch-darkly provides a special `variation` import that can be used in Javascript files such as Components.

If your feature flag is a boolean based flag, you might use it in a function like so:

```js
// /app/components/login-page/component.js

import Component from 'ember-component';
import computed from 'ember-computed';

import { variation } from 'ember-launch-darkly';

export default Component.extend({
  price: computed(function() {
    if (variation('new-pricing-plan')) {
      return 99.00;
    }

    return 199.00;
  })
});
```

If your feature flag is a multivariate based flag, you might use it in a function like so:

```js
// /app/components/login-page/component.js

import Component from 'ember-component';
import computed from 'ember-computed';

import { variation } from 'ember-launch-darkly';

export default Component.extend({
  price: computed(function() {
    switch (variation('new-pricing-plan')) {
      case 'plan-a':
        return 99.00;
      case 'plan-b':
        return 89.00
      case 'plan-c':
        return 79.00
    }

    return 199.00;
  })
});
```

ember-launch-darkly also provides a `variation` computed macro.

```js
// /app/components/login-page/component.js

import Component from 'ember-component';
import computed from 'ember-computed';

import { computedVariation } from 'ember-launch-darkly';

export default Component.extend({
  newPricePlanEnabled: computedVariation('new-pricing-plan')

  price: computed('newPricePlanEnabled', function() {
    if (this.get('newPricePlanEnabled')) {
      return 99.00;
    }

    return 199.00;
  })
});
```

Finally, you can always just inject the Launch Darkly service and use it as you would any other service:

```js
// /app/components/login-page/component.js

import Component from 'ember-component';
import computed from 'ember-computed';
import service from 'ember-service/inject';

export default Component.extend({
  launchDarkly: service(),

  price: computed('launchDarkly.newPricePlan', function() {
    if (this.get('launchDarkly.newPricePlan')) {
      return 99.00;
    }

    return 199.00;
  }),

  discount: computed(function() {
    if (this.get('launchDarkly').variation('apply-discount')) {
      return 0.5;
    }

    return null;
  })
});
```

## Local feature flags

When `local: true` is set in the Launch Darkly configuration, ember-launch-darkly will retrieve the feature flags and their values from `config/environment.js` instead of the Launch Darkly service. This is useful for development purposes so you don't need to set up a new environment in Launch Darkly, your app doesn't need to make a request for the flags, and you can easily change the value of the flags from the browser console.

The local feature flags are defined in `config/environment.js` like so:

```js
let ENV = {
  launchDarkly: {
    local: true,
    localFeatureFlags: {
      'apply-discount': true.
      'new-pricing-plan': 'plan-a'
    }
  }
}
```

When `local: true`, the Launch Darkly feature service is available in the browser console via `window.ld`. The service provides the following helper methods to manipulate feature flags:

```js
> ld.variation('apply-discount') // return the current value of the feature flag

> ld.variation('apply-discount', true) // set the return value of the feature flag
> ld.variation('new-pricing-plan', 'plan-a') // set the return value of the feature flag

> ld.enable('apply-discount') // helper to set the return value to `true`
> ld.disable('apply-discount') // helper to set the return value to `false`

> ld.allFlags() // return the current list of feature flags and their values

> ld.user() // return the user that the client has been initialized with
```

## Test Helpers

### Acceptance Tests

ember-launch-darkly provides a test helper, `withVariation`, to make it easy to turn feature flags on and off in acceptance tests. Simply import the test helper in your test, or `tests/test-helper.js` file.

```js
import 'ember-launch-darkly/test-support/helpers/with-variation';

test( "links go to the new homepage", function () {
  withVariation('apply-discount', true);
  withVariation('new-pricing-plan', 'plan-a');

  visit('/');
  click('a.pricing');
  andThen(function(){
    equal(currentRoute(), 'new.pricing', 'Should be on the new pricing page');
  });
});
```

### Integration Tests

Simply stub the Launch Darkly service in integration tests to control the feature flags

```js
import getOwner from 'ember-owner/get';
import Service from 'ember-service';

let stubService = Service.extend({
  variation(key) {
    if (key === 'new-pricing-page') {
      return 'plan-a';
    }

    return false;
  }
});

moduleForComponent('my-component', 'Integration | Component | my component', {
  integration: true,
  beforeEach() {
    // register the stub service
    this.register('service:launch-darkly', stubService);

    // inject here if you want to be able to inspect/manipulate the service in tests
    this.inject.service('launch-darkly', { as: 'launchDarklyService' });
  }
});
```

## Caveats

### Default variation state

By default a call to `variation` will return `false` if, for some reason, it can't get the true value of the feature flag. Therefore, it's important that variations are always used in the positive, ie, if the flag is enabled, perform new logic and if it's disabled, revert to the existing logic.
