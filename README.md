# axios-hooks

[![Build Status](https://travis-ci.org/simoneb/axios-hooks.svg?branch=master)](https://travis-ci.org/simoneb/axios-hooks)
[![npm version](https://badge.fury.io/js/axios-hooks.svg)](https://badge.fury.io/js/axios-hooks)
[![bundlephobia](https://badgen.net/bundlephobia/minzip/axios-hooks)](https://bundlephobia.com/result?p=axios-hooks)

React hooks for [axios], with built-in support for server side rendering.

## Features

- All the [axios] awesomeness you are familiar with
- Zero configuration, but configurable if needed
- One-line usage
- Super straightforward to use with SSR

## Installation

`npm install axios axios-hooks`

> `axios` is a peer dependency and needs to be installed explicitly

## Quick Start

[![Edit axios-hooks Quick Start](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/2oxrlq8rjr)

```js
import useAxios from 'axios-hooks'

function App() {
  const [{ data, loading, error }, refetch] = useAxios(
    'https://api.myjson.com/bins/820fc'
  )

  if (loading) return <p>Loading...</p>
  if (error) return <p>Error!</p>

  return (
    <div>
      <button onClick={refetch}>refetch</button>
      <pre>{JSON.stringify(data, null, 2)}</pre>
    </div>
  )
}
```

## Documentation

### API

- [useAxios](#useaxiosurlconfig-options)
- [configure](#configure-cache-axios-)
- [serializeCache](#serializeCache)
- [loadCache](#loadcachecache)

### Guides

- [Configuration](#configuration)
- [Manual Requests](#manual-requests)
- [Server Side Rendering](#server-side-rendering)

## API

The package exports one default export and named exports:

`import useAxios, { configure, loadCache, serializeCache } from 'axios-hooks'`

### useAxios(url|config, options)

The main React hook to execute HTTP requests.

- `url|config` - The request URL or [config](https://github.com/axios/axios#request-config) object, the same argument accepted by `axios`.
- `options` - An options object.
  - `manual` ( `false` ) - If true, the request is not executed immediately. Useful for non-GET requests that should not be executed when the component renders. Use the `execute` function returned when invoking the hook to execute the request manually.
  - `useCache` ( `true` ) - Allows caching to be enabled/disabled for the hook. It doesn't affect the `execute` function returned by the hook.

Returns:

`[{ data, loading, error, response }, execute]`

- `data` - The [success response](https://github.com/axios/axios#response-schema) data property (for convenient access).
- `loading` - True if the request is in progress, otherwise False.
- `error` - The [error](https://github.com/axios/.axios#handling-errors) value
- `response` - The whole [success response](https://github.com/axios/axios#response-schema) object.

- `execute([config[, options]])` - A function to execute the request manually, bypassing the cache by default.
  - `config` - Same `config` object as `axios`, which is _shallow-merged_ with the config object provided when invoking the hook. Useful to provide arguments to non-GET requests.
  - `options` - An options object.
    - `useCache` ( `false` ) - Allows caching to be enabled/disabled for this "execute" function.

### configure({ cache, axios })

Allows to provide custom instances of cache and axios.

- `cache` An instance of [lru-cache](https://github.com/isaacs/node-lru-cache)
- `axios` An instance of [axios](https://github.com/axios/axios#creating-an-instance)

### serializeCache()

Dumps the request-response cache, to use in server side sendering scenarios.

Returns:

`Promise<Array>` A serializable representation of the request-response cache ready to be used by `loadCache`

### loadCache(cache)

Populates the cache with serialized data generated by `serializeCache`.

- `cache` The serializable representation of the request-response cache generated by `serializeCache`

## Manual requests

On the client, requests are executed when the component renders using a React `useEffect` hook.

This may be undesirable, as in the case of non-GET requests. By using the `manual` option you can skip the automatic execution of requests and use the return value of the hook to execute them manually, optionally providing configuration overrides to `axios`.

### Example

In the example below we use the `useAxios` hook twice. Once to load the data when the component renders, and once to submit data updates via a `PUT` request configured via the `manual` option.

[![Edit axios-hooks Manual Request](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/axioshooks-manual-request-bq9w4?fontsize=14)

```js
import useAxios from 'axios-hooks'

function App() {
  const [{ data: getData, loading: getLoading, error: getError }] = useAxios(
    'https://api.myjson.com/bins/820fc'
  )

  const [
    { data: putData, loading: putLoading, error: putError },
    executePut
  ] = useAxios(
    {
      url: 'https://api.myjson.com/bins/820fc',
      method: 'PUT'
    },
    { manual: true }
  )

  function updateData() {
    executePut({
      data: {
        ...getData,
        updatedAt: new Date().toISOString()
      }
    })
  }

  if (getLoading || putLoading) return <p>Loading...</p>
  if (getError || putError) return <p>Error!</p>

  return (
    <div>
      <button onClick={updateData}>update data</button>
      <pre>{JSON.stringify(putData || getData, null, 2)}</pre>
    </div>
  )
}
```

## Configuration

Unless provided via the `configure` function, `axios-hooks` uses as defaults:

- `axios` - the default `axios` package export
- `cache` - a new instance of the default `lru-cache` package export, with no arguments

These defaults may not suit your needs:

- you may want a common base url for axios requests
- the default (Infinite) cache size may not be a sensible default

In such cases you can use the `configure` function to provide your custom implementation of both.

> When `configure` is used, it should be invoked once before any usages of the `useAxios` hook

### Example

[![Edit axios-hooks configuration example](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/oqvxw6mpyq)

```js
import { configure } from 'axios-hooks'
import LRU from 'lru-cache'
import Axios from 'axios'

const axios = Axios.create({
  baseURL: 'https://api.myjson.com/'
})

const cache = new LRU({ max: 10 })

configure({ axios, cache })
```

## Server Side Rendering

`axios-hooks` seamlessly supports server side rendering scenarios, by preloading data on the server and providing the data to the client, so that the client doesn't need to reload it.

### How it works

1. the React component tree is rendered on the server
2. `useAxios` HTTP requests are executed on the server
3. the server code awaits `serializeCache()` in order to obtain a serializable representation of the request-response cache
4. the server injects a JSON-serialized version of the cache in a `window` global variable
5. the client hydrates the cache from the global variable before rendering the application using `loadCache`

### Example

[![Edit axios-hooks SSR example](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/v83l3mjq57)

```html
<!-- fragment of the HTML template defining the window global variable -->

<script>
  window.__AXIOS_HOOKS_CACHE__ = {{{cache}}}
</script>
```

```js
// server code for the server side rendering handler

import { serializeCache } from 'axios-hooks'

router.use(async (req, res) => {
  const index = fs.readFileSync(`${publicFolder}/index.html`, 'utf8')
  const html = ReactDOM.renderToString(<App />)

  // wait for axios-hooks HTTP requests to complete
  const cache = await serializeCache()

  res.send(
    index
      .replace('{{{html}}}', html)
      .replace('{{{cache}}}', JSON.stringify(cache).replace(/</g, '\\u003c'))
  )
})
```

```js
// client side code for the application entry-point

import { loadCache } from 'axios-hooks'

loadCache(window.__AXIOS_HOOKS_CACHE__)

delete window.__AXIOS_HOOKS_CACHE__

ReactDOM.hydrate(<App />, document.getElementById('root'))
```

## Promises

axios-hooks depends on a native ES6 Promise implementation to be [supported](http://caniuse.com/promises).
If your environment doesn't support ES6 Promises, you can [polyfill](https://github.com/jakearchibald/es6-promise).

## Credits

`axios-hooks` is heavily inspired by [graphql-hooks](https://github.com/nearform/graphql-hooks),
developed by the awesome people at [NearForm](https://github.com/nearform).

## License

MIT

[axios]: https://github.com/axios/axios
