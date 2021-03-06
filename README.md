[![npm version](https://badge.fury.io/js/mappersmith.svg)](http://badge.fury.io/js/mappersmith)
[![Build Status](https://travis-ci.org/tulios/mappersmith.svg?branch=master)](https://travis-ci.org/tulios/mappersmith)
# Mappersmith

__Mappersmith__ is a lightweight rest client for node.js and the browser. It creates a client for your API, gathering all configurations into a single place, freeing your code from HTTP configurations.

## Table of Contents

1. [Installation](#installation)
1. [Usage](#usage)
  1. [Commonjs](#commonjs)
  1. [Configuring my resources](#resource-configuration)
    1. [Parameters](#parameters)
    1. [Default parameters](#default-parameters)
    1. [Body](#body)
    1. [Headers](#headers)
    1. [Basic Auth](#basic-auth)
    1. [Timeout](#timeout)
    1. [Alternative host](#alternative-host)
    1. [Binary data](#binary-data)
  1. [Promises](#promises)
  1. [Response object](#response-object)
  1. [Middlewares](#middlewares)
  1. [Testing Mappersmith](#testing-mappersmith)
  1. [Gateways](#gateways)
1. [Development](#development)

## <a name="installation"></a> Installation

#### NPM

```sh
npm install mappersmith --save
# yarn add mappersmith
```

#### Browser

Download the tag/latest version from the dist folder.

#### Build from the source

Install the dependencies

```sh
yarn
```

Build

```sh
npm run build
npm run release # for minified version
```

## <a name="usage"></a> Usage

To create a client for your API you will need to provide a simple manifest. If your API reside in the same domain as your app you can skip the `host` configuration. Each resource has a name and a list of methods with its definitions, like:

```javascript
import forge from 'mappersmith'

const github = forge({
  host: 'https://status.github.com',
  resources: {
    Status: {
      current: { path: '/api/status.json' },
      messages: { path: '/api/messages.json' },
      lastMessage: { path: '/api/last-message.json' }
    }
  }
})

github.Status.lastMessage().then((response) => {
  console.log(`status: ${response.data()}`)
})
```

## <a name="commonjs"></a> Commonjs

If you are using _commonjs_, your `require` should look like:

```javascript
const forge = require('mappersmith').default
```

## <a name="resource-configuration"></a> Configuring my resources

Each resource has a name and a list of methods with its definitions. A method definition can have host, path, method, headers, params, bodyAttr, headersAttr and authAttr. Example:

```javascript
const client = forge({
  resources: {
    User: {
      all: { path: '/users' },

      // {id} is a dynamic segment and will be replaced by the parameter "id"
      // when called
      byId: { path: '/users/{id}' },

      // {group} is also a dynamic segment but it has default value "general"
      byGroup: { path: '/users/groups/{group}', params: { group: 'general' } }
    },
    Blog: {
      // The HTTP method can be configured through the `method` key, and a default
      // header "X-Special-Header" has been configured for this resource
      create: { method: 'post', path: '/blogs', headers: { 'X-Special-Header': 'value' } },

      // There are no restrictions for dynamic segments and HTTP methods
      addComment: { method: 'put', path: '/blogs/{id}/comment' }
    }
  }
})
```

### <a name="parameters"></a> Parameters

If your method doesn't require any parameter, you can just call it without them:

```javascript
client.User
  .all() // https://my.api.com/users
  .then((response) => console.log(response.data()))
  .catch((response) => console.error(response.data()))
```

Every parameter that doesn't match a pattern `{parameter-name}` in path will be sent as part of the query string:

```javascript
client.User.all({ active: true }) // https://my.api.com/users?active=true
```

When a method requires a parameters and the method is called without it, __Mappersmith__ will raise an error:

```javascript
client.User.byId(/* missing id */)
// throw '[Mappersmith] required parameter missing (id), "/users/{id}" cannot be resolved'
```

### <a name="default-parameters"></a> Default Parameters

It is possible to configure default parameters for your resources, just use the key `params` in the definition. It will replace params in the URL or include query strings.

If we call `client.User.byGroup` without any params it will default `group` to "general"

```javascript
client.User.byGroup() // https://my.api.com/users/groups/general
```

And, of course, we can override the defaults:

```javascript
client.User.byGroup({ group: 'cool' }) // https://my.api.com/users/groups/cool
```

### <a name="body"></a> Body

To send values in the request body (usually for POST, PUT or PATCH methods) you will use the special parameter `body`:

```javascript
client.Blog.create({
  body: {
    title: 'Title',
    tags: ['party', 'launch']
  }
})
```

By default, it will create a _urlencoded_ version of the object (`title=Title&tags[]=party&tags[]=launch`). If the body used is not an object it will use the original value. If `body` is not possible as a special parameter for your API you can configure it through the param `bodyAttr`:

```javascript
// ...
{
  create: { method: 'post', path: '/blogs', bodyAttr: 'payload' }
}
// ...

client.Blog.create({
  payload: {
    title: 'Title',
    tags: ['party', 'launch']
  }
})
```

__NOTE__: It's possible to post body as JSON, check the [EncodeJsonMiddleware](#encode-json-middleware) below for more information

### <a name="headers"></a> Headers

To define headers in the method call use the parameter `headers`:

```javascript
client.User.all({ headers: { Authorization: 'token 1d1435k' } })
```

If `headers` is not possible as a special parameter for your API you can configure it through the param `headersAttr`:

```javascript
// ...
{
  all: { path: '/users', headersAttr: 'h' }
}
// ...

client.User.all({ h: { Authorization: 'token 1d1435k' } })
```

### <a name="basic-auth"></a> Basic auth

To define credentials for basic auth use the parameter `auth`:

```javascript
client.User.all({ auth: { username: 'bob', password: 'bob' } })
```

The available attributes are: `username` and `password`.
This will set an `Authorization` header. This can still be overridden by custom headers.

If `auth` is not possible as a special parameter for your API you can configure it through the param `authAttr`:

```javascript
// ...
{
  all: { path: '/users', authAttr: 'secret' }
}
// ...

client.User.all({ secret: { username: 'bob', password: 'bob' } })
```

__NOTE__: A default basic auth can be configured with the use of the [BasicAuthMiddleware](#basic-auth-middleware), check the middlewares section below for more information.

### <a name="timeout"></a> Timeout

To define the number of milliseconds before the request times out use the parameter `timeout`:

```javascript
client.User.all({ timeout: 1000 })
```

If `timeout` is not possible as a special parameter for your API you can configure it through the param `timeoutAttr`:

```javascript
// ...
{
  all: { path: '/users', timeoutAttr: 'maxWait' }
}
// ...

client.User.all({ maxWait: 500 })
```

__NOTE__: A default timeout can be configured with the use of the [TimeoutMiddleware](#timeout-middleware), check the middlewares section below for more information.

### <a name="alternative-host"></a> Alternative host

There are some cases where a resource method resides in another host, in those cases you can use the `host` key to configure a new host:

```javascript
// ...
{
  all: { path: '/users', host: 'http://old-api.com' }
}
// ...

client.User.all() // http://old-api.com/users
```

### <a name="binary-data"></a> Binary data

If the data being fetched is in binary form, such as a PDF, you may add the `binary` key, and set it to true. The response data will then be a [Buffer](https://nodejs.org/api/buffer.html) in NodeJS, and a [Blob](https://developer.mozilla.org/sv-SE/docs/Web/API/Blob) in the browser.

```javascript

// ...
{
  report: { path: '/report.pdf', binary: true }
}
// ...

```

## <a name="promises"></a> Promises

__Mappersmith__ does not apply any polyfills, it depends on a native Promise implementation to be supported. If your environment doesn't support Promises, please apply the polyfill first. One option can be [then/promises](https://github.com/then/promise)

In some cases it is not possible to use/assign the global `Promise` constant, for those cases you can define the promise implementation used by Mappersmith.

For example, using the project [rsvp.js](https://github.com/tildeio/rsvp.js/) (a tiny implementation of Promises/A+):

```javascript
import RSVP from 'rsvp'
import { configs } from 'mappersmith'

configs.Promise = RSVP.Promise
```

All `Promise` references in Mappersmith use `configs.Promise`. The default value is the global Promise.

## <a name="response-object"></a> Response object

Mappersmith will provide an instance of its own `Response` object to the promises. This object has the methods:

* `request()` - Returns the original [Request](https://github.com/tulios/mappersmith/blob/master/src/request.js)
* `status()` - Returns the status number
* `success()` - Returns true for status greater than 200 and lower than 400
* `headers()` - Returns an object with all headers, keys in lower case
* `header(name)` - Returns the value of the header
* `data()` - Returns the response data, if `Content-Type` is `application/json` it parses the response and returns an object

## <a name="middlewares"></a> Middlewares

The behavior between your client and the API can be customized with middlewares. A middleware is a function which returns an object with two methods: request and response.

The `request` method receives an instance of the [Request](https://github.com/tulios/mappersmith/blob/master/src/request.js) object and it must return a Request. The method `enhance` can be used to generate a new request based on the previous one.

The `response` method receives a function which returns a `Promise` resolving the [Response](https://github.com/tulios/mappersmith/blob/master/src/response.js). This function must return a `Promise` resolving the Response. The method `enhance` can be used to generate a new response based on the previous one.

You don't need to implement both methods, you can define only the phase you need.

Example:

```javascript
const MyMiddleware = () => ({
  request(request) {
    return request.enhance({
      headers: { 'x-special-request': '->' }
    })
  },

  response(next) {
    return next().then((response) => response.enhance({
      headers: { 'x-special-response': '<-' }
    }))
  }
})
```

The middleware can be configured using the key `middlewares` in the manifest, example:

```javascript
const client = forge({
  middlewares: [ MyMiddleware ],
  resources: {
    User: {
      all: { path: '/users' }
    }
  }
})
```

It can, optionally, receive the `resourceName` and `resourceMethod`, example:

```javascript
const MyMiddleware = ({ resourceName, resourceMethod }) => ({
  /* ... */
})

client.User.all()
// resourceName: 'User'
// resourceMethod: 'all'
```

### Built-in middlewares

#### <a name="encode-json-middleware"></a> EncodeJson

Automatically encode your objects into JSON

```javascript
import EncodeJson from 'mappersmith/middlewares/encode-json'

const client = forge({
  middlewares: [ EncodeJson ],
  /* ... */
})

client.User.all({ body: { name: 'bob' } })
// => body: {"name":"bob"}
// => header: "Content-Type=application/json;charset=utf-8"
```

#### GlobalErrorHandler

Provides a catch-all function for all requests. If the catch-all function returns `true` it prevents the original promise to continue.

```javascript
import GlobalErrorHandler, { setErrorHandler } from 'mappersmith/middlewares/global-error-handler'

setErrorHandler((response) => {
  console.log('global error handler')
  return response.status() === 500
})

const client = forge({
  middlewares: [ GlobalErrorHandler ],
  /* ... */
})

client.User
  .all()
  .catch((response) => console.error('my error'))

// If status != 500
// output:
//   -> global error handler
//   -> my error

// IF status == 500
// output:
//   -> global error handler
```

#### Retry

This middleware will automatically retry GET requests up to the configured amount of retries using a randomization function that grows exponentially. The retry count and the time used will be included as a header in the response.

```javascript
import Retry from 'mappersmith/middlewares/retry'

const client = forge({
  middlewares: [ Retry ],
  /* ... */
})
```

It's possible to configure the header names and parameters used in the calculation.

```javascript
import { setRetryConfigs } from 'mappersmith/middlewares/retry'

// Using the default values as an example
setRetryConfigs({
  headerRetryCount: 'X-Mappersmith-Retry-Count',
  headerRetryTime: 'X-Mappersmith-Retry-Time',
  maxRetryTimeInSecs: 5,
  initialRetryTimeInSecs: 0.1,
  factor: 0.2, // randomization factor
  multiplier: 2, // exponential factor
  retries: 5 // max retries
})
```

#### <a name="basic-auth-middleware"></a> BasicAuth

Automatically configure your requests with basic auth

```javascript
import BasicAuthMiddleware from 'mappersmith/middlewares/basic-auth'
const BasicAuth = BasicAuthMiddleware({ username: 'bob', password: 'bob' })

const client = forge({
  middlewares: [ BasicAuth ],
  /* ... */
})

client.User.all()
// => header: "Authorization: Basic Ym9iOmJvYg=="
```

** The default auth can be overridden with the explicit use of the auth parameter, example:

```javascript
client.User.all({ auth: { username: 'bill', password: 'bill' } })
// auth will be { username: 'bill', password: 'bill' } instead of { username: 'bob', password: 'bob' }
```

#### <a name="timeout-middleware"></a> Timeout

Automatically configure your requests with a default timeout

```javascript
import TimeoutMiddleware from 'mappersmith/middlewares/timeout'
const Timeout = TimeoutMiddleware(500)

const client = forge({
  middlewares: [ Timeout ],
  /* ... */
})

client.User.all()
```

** The default timeout can be overridden with the explicit use of the timeout parameter, example:

```javascript
client.User.all({ timeout: 100 })
// timeout will be 100 instead of 500
```

#### Log

Log all requests and responses. Might be useful in development mode.

```javascript
import Log from 'mappersmith/middlewares/log'

const client = forge({
  middlewares: [ Log ],
  /* ... */
})
```

#### Duration

Automatically adds `X-Started-At`, `X-Ended-At` and `X-Duration` headers to the response.

```javascript
import Duration from 'mappersmith/middlewares/duration'

const client = forge({
  middlewares: [ Duration ],
  /* ... */
})

client.User.all({ body: { name: 'bob' } })
// => headers: "X-Started-At=1492529128453;X-Ended-At=1492529128473;X-Duration=20"
```

#### <a name="csrf-middleware"></a> Csrf

Automatically configure your requests by adding a header with the value of a cookie - If it exists.
The name of the cookie (defaults to "csrfToken") and the header (defaults to "x-csrf-token") can be set as following;

```javascript
import Csrf from 'mappersmith/middlewares/csrf'

const client = forge({
  middlewares: [ Csrf('csrfToken', 'x-csrf-token') ],
  /* ... */
})

client.User.all()
```

## <a name="testing-mappersmith"></a> Testing Mappersmith

Mappersmith plays nice with all test frameworks, the generated client is a plain javascript object and all the methods can be mocked without any problem. However, this experience can be greatly improved with the test library.

The test library has 4 utilities: `install`, `uninstall`, `mockClient` and `mockRequest`

#### install and uninstall

They are used to setup the test library, __example using jasmine__:

```javascript
import { install, uninstall } from 'mappersmith/test'

describe('Feature', () => {
  beforeEach(() => install())
  afterEach(() => uninstall())
})
```

#### mockClient

`mockClient` offers a high level abstraction, it works directly on your client mocking the resources and their methods.

It accepts the methods:

* `resource(resourceName)`, ex: `resource('Users')`
* `method(resourceMethodName)`, ex: `method('byId')`
* `with(resourceMethodArguments)`, ex: `with({ id: 1 })`
* `status(statusNumber)`, ex: `status(204)`
* `response(responseData)`, ex: `response({ user: { id: 1 } })`
* `assertObject()`

Example using __jasmine__:

```javascript
import forge from 'mappersmith'
import { install, uninstall, mockClient } from 'mappersmith/test'

describe('Feature', () => {
  beforeEach(() => install())
  afterEach(() => uninstall())

  it('works', (done) => {
    const myManifest = {} // Let's assume I have my manifest here
    const client = forge(myManifest)

    mockClient(client)
      .resource('User')
      .method('all')
      .response({ allUsers: [{id: 1}] })

    // now if I call my resource method, it should return my mock response
    client.User
      .all()
      .then((response) => expect(response.data()).toEqual({ allUsers: [{id: 1}] }))
      .then(done)
  })
})
```

To mock a failure just use the correct HTTP status, example:

```javascript
// ...
mockClient(client)
  .resource('User')
  .method('byId')
  .with({ id: 'ABC' })
  .status(422)
  .response({ error: 'invalid ID' })
// ...
```

The method `with` accepts the body and headers attributes, example:

```javascript
// ...
mockClient(client)
  .with({
    id: 'abc',
    headers: { 'x-special': 'value'},
    body: { payload: 1 }
  })
  // ...
```

It's possible to use a match function to assert params and body, example:

```javascript
import { m } from 'mappersmith/test'

mockClient(client)
  .with({
    id: 'abc',
    name: m.stringContaining('john'),
    headers: { 'x-special': 'value'},
    body: m.stringMatching(/token=[^&]+&other=true$/)
  })
```

The assert object can be used to retrieve the requests, example:

```javascript
const mock = mockClient(client)
  .resource('User')
  .method('all')
  .response({ allUsers: [{id: 1}] })
  .assertObject()

console.log(mock.mostRecentCall())
console.log(mock.callsCount())
console.log(mock.calls())
```

#### mockRequest

`mockRequest` offers a low level abstraction, very useful for automations.

It accepts the params: method, url, body and response

It returns an assert object

Example using __jasmine__:

```javascript
import forge from 'mappersmith'
import { install, uninstall, mockRequest } from 'mappersmith/test'

describe('Feature', () => {
  beforeEach(() => install())
  afterEach(() => uninstall())

  it('works', (done) => {
    mockRequest({
      method: 'get',
      url: 'https://my.api.com/users?someParam=true',
      response: {
        body: { allUsers: [{id: 1}] }
      }
    })

    const myManifest = {} // Let's assume I have my manifest here
    const client = forge(myManifest)

    client.User
      .all()
      .then((response) => expect(response.data()).toEqual({ allUsers: [{id: 1}] }))
      .then(done)
  })
})
```

A more complete example:

```javascript
// ...
mockRequest({
  method: 'post',
  url: 'http://example.org/blogs',
  body: 'param1=A&param2=B', // request body
  response: {
    status: 503,
    body: { error: true },
    headers: { 'x-header': 'nope' }
  }
})
// ...
```

It's possible to use a match function to assert the body and the URL, example:

```javascript
import { m } from 'mappersmith/test'

mockRequest({
  method: 'post',
  url: m.stringMatching(/example\.org/),
  body: m.anything(),
  response: {
    body: { allUsers: [{id: 1}] }
  }
})
```

Using the assert object:

```javascript
const mock = mockRequest({
  method: 'get',
  url: 'https://my.api.com/users?someParam=true',
  response: {
    body: { allUsers: [{id: 1}] }
  }
})

console.log(mock.mostRecentCall())
console.log(mock.callsCount())
console.log(mock.calls())
```

#### Match functions

`mockClient` and `mockRequest` accept match functions, the available built-in match functions are:

```javascript
import { m } from 'mappersmith/test'

m.stringMatching(/something/) // accepts a regexp
m.stringContaining('some-string') // accepts a string
m.anything()
m.uuid4()
```

A match function is a function which returns a boolean, example:

```javascript
mockClient(client)
  .with({
    id: 'abc',
    headers: { 'x-special': 'value'},
    body: (body) => body === 'something'
  })
```

__Note__:  
`mockClient` only accepts match functions for __body__ and __params__  
`mockRequest` only accepts match functions for __body__ and __url__  

## <a name="gateways"></a> Gateways

Mappersmith has a pluggable transport layer and it includes by default three gateways: xhr, http and fetch. Mappersmith will pick the correct gateway based on the environment you are running (nodejs or the browser).

You can write your own gateway, take a look at [XHR](https://github.com/tulios/mappersmith/blob/master/src/gateway/xhr.js) for an example. To configure, import the `configs` object and assign the gateway option, like:

```javascript
import { configs } from 'mappersmith'
configs.gateway = MyGateway
```

It's possible to globally configure your gateway through the option `gatewayConfigs`.

### HTTP

When running with node.js you can configure the `configure` callback to further customize the `http/https` module, example:

```javascript
import fs from 'fs'
import https from 'https'
import { configs } from 'mappersmith'

const key = fs.readFileSync('/path/to/my-key.pem')
const cert =  fs.readFileSync('/path/to/my-cert.pem')

configs.gatewayConfigs.HTTP = {
  configure() {
    return {
      agent: new https.Agent({ key, cert })
    }
  }
}
```

The new configurations will be merged. `configure` also receives the `requestParams` as the first argument. Take a look [here](https://github.com/tulios/mappersmith/blob/master/src/mappersmith.js) for more options.

### XHR

When running in the browser you can configure `withCredentials` and `configure` to further customize the `XMLHttpRequest` object, example:

```javascript
import { configs } from 'mappersmith'
configs.gatewayConfigs.XHR = {
  withCredentials: true,
  configure(xhr) {
    xhr.ontimeout = () => console.error('timeout!')
  }
}
```

Take a look [here](https://github.com/tulios/mappersmith/blob/master/src/mappersmith.js) for more options.

### Fetch

__Mappersmith__ does not apply any polyfills, it depends on a native `fetch` implementation to be supported. It is possible assign the fetch implementation used by Mappersmith:

```javascript
import { configs } from 'mappersmith'
configs.fetch = fetchFunction
```

Fetch is not used by default, you can configure it through `configs.gateway`.

```javascript
import FetchGateway from 'mappersmith/gateway/fetch'
import { configs } from 'mappersmith'

configs.gateway = FetchGateway

// Extra configurations, if needed
configs.gatewayConfigs.Fetch = {
  credentials: 'same-origin'
}
```

Take a look [here](https://github.com/tulios/mappersmith/blob/master/src/mappersmith.js) for more options.

## <a name="development"></a> Development

### Running unit tests:

```sh
yarn test:browser
yarn test:node
```

### Running integration tests:

```sh
yarn integration-server &
yarn test:browser:integration
yarn test:node:integration
```

### Running all tests

```sh
node spec/integration/server.js &
yarn test
```

## Compile and release

```sh
NODE_ENV=production yarn build
```

## Contributors

Check it out!

https://github.com/tulios/mappersmith/graphs/contributors

## License

See [LICENSE](https://github.com/tulios/mappersmith/blob/master/LICENSE) for more details.
