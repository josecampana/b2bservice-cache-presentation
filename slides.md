---
# try also 'default' to start simple
# theme: default
layout: cover
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
#background: https://source.unsplash.com/collection/94734566/1920x1080
background: https://concepto.de/wp-content/uploads/2018/08/cache3-e1534944934335.jpg
# apply any windi css classes to the current slide
# class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Using @b2b/service cache system


  Learn more at [@b2b/servive](https://github.com/ingka-group-digital/b2b-shared-nodejs/tree/main/packages/b2b-service)
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: fade-out
# use UnoCSS
# css: unocss
fonts:
  # basically the text
  sans: 'Robot'
  # use with `font-serif` css class from windicss
  serif: 'Robot Slab'
  # for code blocks, inline code, etc.
  mono: 'Fira Code'
---

# Redis cache in BOX

Working with @b2b/service cache system

<!--
Thank you for coming. I'm going to talk about how we are adding a Redis cache in our API projects that uses @b2b/service
-->

---

# Available cache types (store types)

- Internally uses [@b2b/store module](https://github.com/ingka-group-digital/b2b-shared-nodejs/tree/main/packages/b2b-store) as a cache store
- Two store implementations:

  - Memory in the instance of your app (LRU)
  - [Redis server (https://redis.io/)](https://redis.io/)

Warning: if you deploy more than one instance of your app, you will have alignment problems in the cache content if you use memory cache.

<!--
Before having a Redis server, some of our APIs were using memory cache.
Problems of this: if you have more than 1 instance of your app deployed, each instance has its own memory cache, they are not synched so the result could vary depending on which instance is resturning the information

By using the module @b2b/store we can switch easily between memory and redis.
-->

---

# Enabling the cache

At <kbd>src/index.js</kbd> set the cache property to type auto <kbd>{ type: 'auto'}</kbd>

```javascript
const service = require('@b2b/service')();
service.cache = { type: 'auto' };
service.listen();
```

If type === 'auto'

- look for for `config/default.js`.redis.host value
  - found it :) => it will create a Redis store
  - not found :( => it will create a memory store

<!--
- @b2b/store is integrated into @b2b/service so enable the cache (memory or redis) is quite easy
- instead of configure the projects to use memory cache, we configured them in auto mode thinking in the future
- the auto mode will check your config object, looking for the field redis.host to setup a Redis cache or a memory cache
-->

---

# ENV variables

- REDIS_HOST
- REDIS_PORT (default 6379)
- REDIS_PASSWORD (optional)
- REDIS_DB (default 0)
- CACHE_TTL (in milliseconds)

<!--
- in BOX we defined the following environment variables that are common for all of our projects
- you could use another names in non BOX projects. Just be sure you are using it in your config file
-->

---

# Config file

A partial example of the <kbd>config/default.js</kbd> file content

<div grid="~ cols-2 gap-4">
<div>

```javascript
const {
  REDIS_HOST,
  REDIS_PORT,
  REDIS_PASSWORD,
  REDIS_DB,
  CACHE_TTL: ttl = 2 * 60 * 60 * 1000,
  PORT: port = 3002,
  LOG_LEVEL: logLevel = 'info',
  TRANSLATION_TAG: tag = 'my_module',
} = process.env;
```

</div>
<div>

```javascript
module.exports = {
  logLevel,
  redis: {
    host: REDIS_HOST,
    port: REDIS_PORT,
    password: REDIS_PASSWORD,
    db: REDIS_DB,
  },
  cache: {
    ttl,
  },
  translation: {
    tag,
  },
  ...
};
```

</div>
</div>

This links the env vars with the config needed by @b2b/service

---

# Using the cache in your controller

In your controller, get the cache object from the request:

```javascript

const controller = async (req, res, next) => {
  const { cache } = req;
  const key = 'cache_key_to_retrieve';

  const data = await cache.get(key);

  if(!data){
    //miss cache
    ...
    await cache.set(key, newData);
    // 3rd parameter of cache.set is the TTL.
    // As it is not specified, it will use the one from Cache-control header or the config's cache.ttl
  }
}

```

---

# Considerations

- The Redis server is shared between different apps
- You could only have access to the keys of your app (\*)

```javascript
const key = 'cache_key_to_retrieve';
const data = await cache.get(key); //it is accessing to myappname.cache_key_to_retrieve
```

- Last sentence is not entirely true: you can delete the whole Redis cache by using `cache.prune()`

(\*) when you define a key to store your object in cache, @b2b/service adds a prefix to it. This prefix is the name located at the `package.json`

<!--
- cache object is injected in the request object by a middleware
- this middleware adds a prefix to your keys so you could access only to the app keys
-->

---

# Controlling the cache on the request

[Header 'cache-control' (https://mzl.la/3TmC1d6)](https://mzl.la/3TmC1d6)

- no-cache: do not use cached data
- no-store: do not refresh the current cache
- max-age: ttl to store the data when refreshing the cache (in seconds)

### examples

```bash
# do not use cache to retrieve data. Store the results with a new TTL of 1h
curl -X 'GET' \
  'https://dev.ecom.ifb.ingka.com/checkout/api/v1/translations/es/es' \
  -H 'accept: application/json' \
  -H 'cache-control: no-cache,max-age=3600'
```

```bash
# do not use cache to retrieve data. Do not store the results in cache for future calls
curl -X 'GET' \
  'https://dev.ecom.ifb.ingka.com/checkout/api/v1/translations/es/es' \
  -H 'accept: application/json' \
  -H 'cache-control: no-cache,no-store'
```

<!--
- cache object is injected in the request object by a middleware
- you don't need to add extra code in your app to check the cache control headers (the middleware overrides the cache functions)
-->

---

# Controlling the cache on the request (ii)

Header 'x-cache-request'

this is a custom header that will retrive from cache the entire response of your **GET ENDPOINT** when is set to true

### example:

```bash
curl -X 'GET' \
  'https://dev.ecom.ifb.ingka.com/checkout/api/v1/translations/es/es' \
  -H 'accept: application/json' \
  -H 'x-cache-request: true'
```

---

# TTL when storing data with cache.set()

The following priority for TTL values (Time To Live) will be followed:

- the TTL parameter on set method

```javascript
await cache.set(key, data, myTTL);
```

- the value at max-age (cache-control header);

```bash
curl -X 'GET' \
  'https://dev.ecom.ifb.ingka.com/checkout/api/v1/translations/es/es' \
  -H 'accept: application/json' \
  -H 'cache-control: max-age=3600'
```

- the value at `config/default.js`.cache.ttl

```javascript
module.exports = {
  cache: {
    ttl: process.env.CACHE_TTL || 2 * 60 * 60 * 1000,
  },
  ...
};
```

---

# Available cache methods

- await cache.get(key):
  - hit!: the data object stored by a key
  - miss: undefined
- await cache.set(key, data, ttl): store a data object by a key with an optional TTL
- await cache.keys(): retrive a list of available keys
- await cache.remove(key): remove the data object stored by key from cache
- await cache.removeAll(): remove all data objects of my app

Remember:

- The Redis service is shared between apps
- An @b2b/service app could not access the keys of another app.
- cache.prune() could flush the entire cache

---

# The middleware

@b2b/service includes an express middleware that manages all these stuff: [/src/express/cache.js](https://github.com/ingka-group-digital/b2b-shared-nodejs/blob/main/packages/b2b-service/src/express/cache.js)

- Creates the redis or lru store (aka cache) depending on the config
- Generates the key prefix for all the keys of your app
- Take care about cache-control and x-cache-request headers
- Take care that you are not accessing keys outsite your app
- Injects the cache into the request object to get it available at controllers

---

# Gimme the prize

<div grid="~ cols-2 gap--4">
<div>
Start using it by adding the following line into the .npmrc file of your project:

```bash
@b2b:registry=https://europe-npm.pkg.dev/ingka-b2b-components-prod/b2b/
```

And install @b2b/service:

```bash
npm i @b2b/service@latest
```

If you are working on an existing express project without @b2b/service, you could use @b2b/store and [replicate the logic of the cache middleware](https://github.com/ingka-group-digital/b2b-shared-nodejs/blob/main/packages/b2b-service/src/express/cache.js)

```bash
npm i @b2b/store@latest
```

Note: if you are using @b2b/service you don't need to install @b2b/store

</div>
<div>

<iframe width="300" height="200" src="https://www.youtube.com/embed/GT1k3-fErCM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>
</div>

---

# I LOVE REDIS - IN REDIS WE TRUST

![](https://www.anfibiosexoticos.com/wp-content/uploads/2020/05/hipnosapo-min.jpg)

Use @b2b modules ;-)
