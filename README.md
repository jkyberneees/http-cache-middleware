# http-cache-middleware
High performance connect-like HTTP cache middleware for Node.js. 

> Uses `cache-manager` as caching layer, so multiple
storage engines are supported, i.e: Memory, Redis, ... https://www.npmjs.com/package/cache-manager

## Install
```js 
npm i http-cache-middleware
```

## Usage
```js
const middleware = require('http-cache-middleware')()
const service = require('restana')()
service.use(middleware)

service.get('/cache-on-get', (req, res) => {
  setTimeout(() => {
    // keep response in cache for 1 minute if not expired before
    res.setHeader('x-cache-timeout', '1 minute')
    res.send('this supposed to be a cacheable response')
  }, 50)
})

service.delete('/cache', (req, res) => {
  // ... the logic here changes the cache state

  // expire the cache keys using pattern
  res.setHeader('x-cache-expire', '*/cache-on-get')
  res.end()
})

service.start(3000)
```
## Redis cache
```js
// redis setup
const CacheManager = require('cache-manager')
const redisStore = require('cache-manager-ioredis')
const redisCache = CacheManager.caching({
  store: redisStore,
  db: 0,
  host: 'localhost',
  port: 6379,
  ttl: 30
})

// middleware instance
const middleware = require('http-cache-middleware')({
  stores: [redisCache]
})
```

## Why cache? 
> Because caching is the last mile for low latency distributed systems!

Enabling proper caching strategies will drastically reduce the latency of your system, as it reduces network round-trips, database calls and CPU processing.  
For our services, we are talking here about improvements in response times from `X ms` to `~2ms`, as an example.

### Enabling cache for service endpoints
Enabling an response to be cached just requires the 
`x-cache-timeout` header to be set:
```js
res.setHeader('x-cache-timeout', '1 hour')
```
> Here we use the [`ms`](`https://www.npmjs.com/package/ms`) package to convert timeout to seconds. Please note that `millisecond` unit is not supported!  

Example on service using `restana`:
```js
service.get('/numbers', (req, res) => {
  res.setHeader('x-cache-timeout', '1 hour')

  res.send([
    1, 2, 3
  ])
})
```

### Invalidating caches
Services can easily expire cache entries on demand, i.e: when the data state changes. Here we use the `x-cache-expire` header to indicate the cache entries to expire using a matching pattern:
```js
res.setHeader('x-cache-expire', '*/numbers')
```
> Here we use the [`matcher`](`https://www.npmjs.com/package/matcher`) package for matching patterns evaluation.

Example on service using `restana`:
```js
service.patch('/numbers', (req, res) => {
  // ...

  res.setHeader('x-cache-expire', '*/numbers')
  res.send(200)
})
```

### Custom cache keys
Cache keys are generated using: `req.method + req.url`, however, for indexing/segmenting requirements it makes sense to allow cache keys extensions.  

For doing this, we simply recommend using middlewares to extend the keys before caching checks happen:
```js
service.use((req, res, next) => {
  req.cacheAppendKey = (req) => req.user.id // here cache key will be: req.method + req.url + req.user.id  
  return next()
})
```
> In this example we also distinguish cache entries by `user.id`, very common case!

### Disable cache for custom endpoints
You can also disable cache checks for certain requests programmatically:
```js
service.use((req, res, next) => {
  req.cacheDisabled = true
  return next()
})