# Static Prerendering

To further improve the performance of your site, the XDN can be configured to prerender predefined parts (pages, APIs…) of your sites which will be cached on the edge. What you choose to prerender is entirely up to you and the needs of your application. This feature is known as **Static Prerendering** and is especially useful for large, complex sites that have too many URLs to prerender without incurring exceptionally long build times.

Static Prerendering gives your site the speed benefits of a static site by sending requests to your application code and caching the result right after your site is deployed.

> By default, the XDN prerenders a maximum of 200 URLs at a time. This can create significant additional load on your APIs at the time of deployment. - See [Concurrency and Limits](#section_concurrency_and_limits)

Configuring certain parts of your site for prerendering is done in the `xdn.config.js` file which is a regular Node.js module:

```js
// File: xdn.config.js
const { Router } = require('@xdn/core/router');

// The `prerender()` function receives an array of `PrerenderRequest` objects
module.exports = new Router().prerender(PrerenderRequest); 
```

##  Prerendering Paths

To specify the paths to prerender, use the `prerender()` function on the `Router` object:

```js
// File: xdn.config.js
const { Router } = require('@xdn/core/router');

// List of paths to prerender
const pathsToBePrerendered = [
  // HTML Pages
  { path: '/' },
  { path: '/categories/mens' },
  { path: '/categories/mens/shirts' },
  { path: '/categories/mens/pants' },
  { path: '/categories/womens' },
  { path: '/categories/womens/shirts' },
  { path: '/categories/womens/pants' },
];

// `prerender()` is passed the `pathsToBePrerendered`
module.exports = new Router().prerender(pathsToBePrerendered);
```

The `prerender()` function is passed `pathsToBePrerendered`, an array of `PrerenderRequest` objects marked for prerendering.

## Prerendering API responses

Static Prerendering does not apply to pages on your site alone. It is also possible to prerender API responses:

```js
// File: xdn.config.js
const { Router } = require('@xdn/core/router');

// List of API Responses to prerender
const apiResponsesToBePrerendered = [
  { path: '/api/index.json' },
  { path: '/categories/mens.json' },
  { path: '/categories/mens/shirts.json' },
  { path: '/categories/mens/pants.json' },
  { path: '/categories/womens.json' },
  { path: '/categories/womens/shirts.json' },
  { path: '/categories/womens/pants.json' },
];

// `prerender()` is passed the `apiResponsesToBePrerendered`
module.exports = new Router().prerender(apiResponsesToBePrerendered);
```

## Prerendering Async Paths

Hard-coding the list of paths to prerender as in *Prerendering Paths* is useful and straightforward when you know the paths in advance. The XDN can be configured to prerender dynamic paths that can not be predetermined in advance.

```js
// File: xdn.config.js
const { Router } = require('@xdn/core/router');

// List of Async Paths to be prerendered
async function asyncPathsToBePrerendered() {
  // Fetch the list of paths
  const paths = await fetchCategoryPathsFromAPI();

  // Signature of paths remains an array of objects
  // [ {path1}, {path2}, {path3}, ...rest ]
  return paths.map(path => ({ path }))
}

// `prerender()` is passed the `asyncPathsToBePrerendered`
module.exports = new Router().prerender(asyncPathsToBePrerendered);
```

In this case, the `prerender()` function is passed an async function `asyncPathsToBePrerendered()` that returns a list of paths with the same signature - an array of `PrerenderRequest` object(s).

## Prerendering API Calls

Prerendering API calls ensures that client-side navigation is as fast as possible. However, how this is achieved is different in each framework. For example, Next.js embeds a build ID in API URLs to ensure that the client receives responses from the correct version of the back end.

```js
// File: xdn.config.js
const { Router } = require('@xdn/core/router');
const { nextRoutes } = require('@xdn/next');
const { existsSync, readFileSync } = require('fs');
const { join } = require('path');

// Read the Next.js build ID from '.next/BUILD_ID'
const buildIdPath = join(process.cwd(), '.next', 'BUILD_ID');

function getPrerenderRequests() {
  // List of paths to prerender
  const prerenderRequests = [
    { path: '/' },
    { path: '/categories/mens' },
    { path: '/categories/mens/shirts' },
    { path: '/categories/mens/pants' },
    { path: '/categories/womens' },
    { path: '/categories/womens/shirts' },
    { path: '/categories/womens/pants' },
  ];

  // Check to see if the buildIdPath exists
  if(existsSync(buildIdPath)) {
    // Read the buildId from the buildIdPath
    const buildId = readFileSync(buildIdPath, 'utf8');
    
    // Get the API requests from the page URL
    const apiPaths = prerenderRequests.map(path => ({ path: `/data/${buildId}${path}.json` }));

    // Add the generated apiPaths to prerenderRequests
    prerenderRequests.push(...apiPaths)
  }

  return prerenderRequests;
}

module.exports = new Router().prerender(getPrerenderRequests).use(nextRoutes);
```

## Defining Paths via an Environment Variable

The paths marked for pre-rendering can be defined based on the running environment:

```js
// File: xdn.config.js
const { Router } = require('@xdn/core/router');

// List of Paths to be prerendered based on the environment
async function asyncPathsToBePrerendered() {
  // Define the list of paths to prerender in the XDN Developer Console.
  const paths = process.env.PRERENDER_PATHS.split(/\n/);

  // Signature of paths remains an array of objects
  // [ {path1}, {path2}, {path3}, ...rest ]
  return paths.map(path => ({ path }))
}

module.exports = new Router().prerender(envPathsToBePrerendered);
```

## Advanced Configuration: Custom Cache Keys

The edge cache can be split by `cookies` or `headers` using a `CustomCacheKey`, in which case you will need to include the `cookie` or `header` values in the preload configuration.

For example, if you split the cache by a language `cookie`:

```js
// xdn.config.js
const { Router } = require('@xdn/core/router');

module.exports = new Router().prerender([
  // German
  { path: '/categories/mens', headers: { cookie: 'language=de' } },
  { path: '/categories/mens/shirts', headers: { cookie: 'language=de' } },
  { path: '/categories/mens/pants', headers: { cookie: 'language=de' } },
  { path: '/categories/womens', headers: { cookie: 'language=de' } },
  { path: '/categories/womens/shirts', headers: { cookie: 'language=de' } },
  { path: '/categories/womens/pants', headers: { cookie: 'language=de' } },

  // English
  { path: '/categories/mens', headers: { cookie: 'language=en' } },
  { path: '/categories/mens/shirts', headers: { cookie: 'language=en' } },
  { path: '/categories/mens/pants', headers: { cookie: 'language=en' } },
  { path: '/categories/womens', headers: { cookie: 'language=en' } },
  { path: '/categories/womens/shirts', headers: { cookie: 'language=en' } },
  { path: '/categories/womens/pants', headers: { cookie: 'language=en' } },
]);
```

You need to include the language cookie in the preload configuration:

```js
//File: xdn.config.js

// Import the CustomCacheKey object
const { Router, CustomCacheKey } = require('@xdn/core/router');

module.exports = new Router().prerender([
  /* paths to prerender, split by cookies or headers */
]).get('/categories/:slug*', ({ cache }) => {
  cache({
    key: new CustomCacheKey().addCookie('language'),
    edge: { maxAgeSeconds: 60 * 60 * 24, staleWhileRevalidate: 60 * 60 * 24 * 365 }
  });
});
```

##  Concurrency and Limits

There can be a significant additional load on your APIs when your site is been deployed as a result of how the XDN prerenders URLs. By default it will prerender a maximum of 200 URLs concurrently with the following limits imposed based on your tier:

| Tier       | Concurrency | Total number of requests |
| ---------- | ----------- | ------------------------ |
| ENTERPRISE | 200         | 25,000 per deployment    |
| FREE       | 10          | 100 per deployment       |

\
You can lower the limit by setting the [prerenderConcurrency](/guides/xdn_config#section_prerenderconcurrency) property in `xdn.config.js`

## Viewing Prerendering Results in the XDN Developer Console

When you deploy a new version of your site, you can view the progress and results of prerendering from the deployment
view in XDN Developer Console:

![progress](/images/static-prerendering/progress.png)

This section updates in real time as pages are prerendered and will show you any errors that occur. If an error occurs, more information can be found in the build logs.
