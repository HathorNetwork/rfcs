- Feature Name: soft_config_reload
- Start Date: 2023-08-21
- Author: André Carneiro <andre.carneiro@hathor.network>

# Summary
[summary]: #summary

A change to the configuration system to allow changing the configuration without stopping the headless instance.

# Motivation
[motivation]: #motivation

The configuration of the headless is exposed to the source code as a module, and as such it is locked by Node's cache.
We may require some changes to the config without stopping the headless instance, e.g. addind a new seed, changing the network, etc., but with the current system it cannot be done.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Configuration system

### How it works

#### When running locally

The `config.js` file is created by the dev and he can alter the contents as needed.
The dev is responsible for the contents and if they follow the headless requirements.

Once the instance starts, it will load the `config.js` as a module and node's cache system will save this state.
From this point on, any calls that require reading the configuration will read from cache, so writting to the file will have no effect on the running instance.

#### When running in docker container

During the docker build a `config.js` file is created, this file does not have a configuration but instead loads them from the environment variables and then exports the configuration object.
This means that once the wallet starts, the environment variables will be read and the resulting config will be cached the same way as running locally.

### Changes to configuration system

We will change all instances that import `config.js` directly to import a `setting.js` instead, this file will export the methods:
- `setupConfig`: dynamically import the config module and update the configuration singleton.
    - Should be run only once
- `getConfig`: will return the configuration singleton.
    - If the singleton is not present, throw an error
- `reloadConfig`: will erase the configuration singleton, clear any cache on it, then reload the configuration.
    - Will perform checks on the new config as well, to ensure the wallet is running as expected.

This way the dev running locally can continue using the `config.js` module without changes, and the docker configuration will also not change.


## Settings module

The settings module will hold a cache for the configuration, this cache will be a singleton instance (i.e. a simple `_config` object declared on the file).


### Setup config

Since all parts of the wallet will access the config through the settings module it has to be the first thing on the wallet to load since even the server startup depends on the config.

The setup will import the config module, update the singleton and set the value of a variable called `_started` to `true`.
`_started` is initialized with `false` and its function is to signal that the configuration has already been setup, so we don't call this method again.

The reason we can only call this method once is that when called it will import the config and update the singleton ignoring any possible changes to the config.
So if called again it could leave the wallet in an impossible state.


### Get config

Should just check that `_config` is valid and return it.
If `_config` is not valid, it should throw an error, indicating that the configuration cannot be read at the moment (possibly during a reload).
The error it throws should be catched by a middleware which will return `503 (Service Unavailable)` to the caller with the reason for unavailability.

The reload should be a fast method but we cannot allow other calls to be made during the reload since the service can be in an inconsistent state.

### Reload config

This method will save the current config state, delete the node module cache on the `config.js` file then check for the differences on the configuration, some actions may need to be taken depending on the differences.

The action to be takes depends on the severity of the change, we should go down this list and start the appropriate action.

- Changes in `http_bind_address`, `http_port`, `http_api_key`, `consoleLevel`, `httpLogFormat`, `enabled_plugins`, `plugin_config`
    - This are critical changes and require a full restart of the service.
    - We should raise a custom error so that a middleware can return to the user a 200 request then shutdown the service with `process.exit(0)`.
- Changes in `seed` and `multisig`
    - Check that all previous entries are the same.
        - The multisig should check for changes in all subkeys (pubkeys, numSignatures, total)
        - If there was any change in previous entries we should stop all wallets.
        - If there was only new entries, we should do nothing.
    - We should check that all seed values are valid seeds.
        - If an invalid seed is found, we should raise an error to stop the service.
- Changes in `network`, `server`
    - Stop all wallets, then change the configuration using wallet-lib's config.
- Changes in `tokenUid`, `gapLimit`, `connectionTimeout`
    - Stop all wallets, these will take effect when the wallet is started again.
- Changes in `txMiningUrl`, `atomicSwapService`, `txMiningApiKey`
    - Run the `initHathorLib` method in `app.js` to configure these.
- Changes in `confirmFirstAddress`, `allowPassphrase`
    - Do nothing.

The stopped wallets should also be removed from `initializedWallets` on `src/services/wallets.service.js`.
Although it is a bad practice to remove a listener created by another module ([reference](https://nodejs.dev/en/api/v20/events/#emitterremovealllistenerseventname)), we will use the `removeAllListeners` method on the wallet and `wallet.conn` to avoid memory leaks.

## API

### POST /reload-config

This api will run the `reloadConfig` module and return 200.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Dynamic import

In NodeJS we can use [import() expressions](https://nodejs.org/api/esm.html#import-expressions) to dynamically import a module.
Although all rules and managing operation for static imports still apply, dynamic imports are only evaluated when needed (as per the [documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import)).
This allows us to import the config module when needed.

One of the rules that we need to override is the native module cache which would make subsequent calls to dynamically import the config module return the cached value instead of evaluating the module again.

```js
async function importConfig() {
  // The default extraction is because the actual content of the config is inside the default export of the module.
  return (await import(configPath)).default;
}
```

## NodeJS module cache

We interact with the [require cache](https://nodejs.org/api/modules.html#requirecache) to ensure we can dynamically load the `config.js` as a module and the get current value of the config.
Since the `config.js` is imported as a module it falls into a cache table, the key to the cache table is the fullpath of the module entrypoint if its a relative import.
We can get the path used by NodeJS as cache key with [require.resolve](https://nodejs.org/api/modules.html#requireresolverequest-options).

With this information we know we can use the following to invalidate NodeJS cache on the config.js module.

```js
delete require.cache[require.resolve('./config.js')]
```

## Scanning config changes

To avoid any errors with object references we should start the `reloadConfig` method by using [lodash.cloneDeep](https://lodash.com/docs/4.17.15#cloneDeep) on the current `_config` and dynamically importing the current config module.
Then we can make comparisons on the old and new config modules and take appropriate action depending on the changes found.

## Handling errors with middlewares

Middlewares can be configured to run on specific routes, on routers or on the entire application, since the configuration errors are not wallet specific we should use the middleware on the entire application.
To do so we can use the `createApp` method on `src/app.js` to add the middleware to the app.

The [guide](https://expressjs.com/en/guide/error-handling.html) on error handlers tells us the middleware will receive `(error, request, response, next)` where the `error` is the error instance we will intercept.
If the error is not one of the types we want to handle we can just return `next(err)` to invoke the next error handler, it no custom handlers catch the error the default one will, returning `500 (Internal Server Error)` to the user.

```js
function ConfigErrorHandler(err, req, res, next) {
  if (err instanceof NonRecoverableConfigChangeError) {
    res.status(200);
    res.send({ success: false, error: 'A non recoverable change in the config was made, the service will shutdown.' });
    process.exit(0);
  } else if (err instanceof UnavailableConfigError) {
    res.status(503);
    return res.send({ success: false, error: 'Service currently unavailable.' });
  }

  return next(err);
}
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Always read from current config state

When performing a read on any config key, read from the actual source (either envvar or file).
This alternative can have a very poor performance but it is not the reason it was discarded as an option.
There are some special keys that have an impact on the wallet, e.g. network, if we change the network, we should stop all wallets and then reconnect expecting the server to be of the new network.
If we have no way of reacting to changes on the config, we cannot safely discard the possibility that a breaking change was made.

# Future possibilities
[future-possibilities]: #future-possibilities

## Remake the configuration system

The designed system aims to have as little problems as possible with the current configuration system, but we should consider changing the configuration to a less indirect system, without a `config.js` module for instance so the configuration can work the same way locally and in docker.
The configuration is also very error prone for new developers, we should create a type system so any errors on the config can be found as early as possible, maybe even with a static evaluator.

## Standard error response

We should use a common standard for error responses on our service, a good start would be the [RFC7807](https://datatracker.ietf.org/doc/html/rfc7807) standard.

# Task breakdown

- [ ] Create settings module (1 dev day)
- [ ] Create middlewares to handle errors from the settings module (1 dev day)
- [ ] Refactor the headless to use the new settings module instead of config (1 dev day)
- [ ] Refactor the tests to use the new settings module instead of the config (1 dev day)
- [ ] Create tests for the settings module (0.5 dev day)
