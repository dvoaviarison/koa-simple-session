# koa-simple-session

[![Build Status](https://travis-ci.org/vicanso/koa-simple-session.svg?style=flat-square)](https://travis-ci.org/vicanso/koa-simple-session)
[![Coverage Status](https://img.shields.io/coveralls/vicanso/koa-simple-session/master.svg?style=flat)](https://coveralls.io/r/vicanso/koa-simple-session?branch=master)
[![npm](http://img.shields.io/npm/v/koa-simple-session.svg?style=flat-square)](https://www.npmjs.org/package/koa-simple-session)
[![Github Releases](https://img.shields.io/npm/dm/koa-simple-session.svg?style=flat-square)](https://github.com/vicanso/koa-simple-session)

Session middleware for koa 2.x, easy use with [reids](https://github.com/vicanso/koa-simple-redis), supports readonly session (use Object.freeze).

This middleware will only set a cookie when a session is manually set. Each time the session is modified (and only when the session is modified), it will reset the cookie and session.


## Installation

```bash
$ npm install koa-simple-session
``` 

## Examples

```js
'use strict';
const Koa = require('koa');
const Redis = require('koa-simple-redis');
const session = require('koa-simple-session');
const app = new Koa();

function get(ctx) {
  const session = ctx.session;
  session.count = session.count || 0;
  session.count++;
  ctx.body = session.count;
}

function remove(ctx) {
  ctx.session = null;
  ctx.body = 0;
}

function regenerate(ctx) {
  get(ctx);
  return ctx.regenerateSession().then(() => {
    get(ctx);
  });
}

function freeze(ctx) {
  // the session is not sync to redis
  ctx.session.user = {
    a: 'b'
  };
  Object.freeze(ctx.session);
  ctx.body = ctx.session.user;
}

app.name = 'koa-session-test';
app.outputErrors = true;
app.keys = ['keys', 'keykeys'];
app.proxy = true;

app.use(session({
  store: new Redis(),
}));

app.use(ctx => {
  switch (ctx.path) {
    case '/get':
      get(ctx);
      break;
    case '/remove':
      remove(ctx);
      break;
    case '/freeze':
      freeze(ctx);
      break;
    case '/regenerate':
      return regenerate(ctx);
  }
});

app.listen(3000);
```

* After adding session middleware, you can use `this.session` to set or get the sessions.
* Setting `this.session = null;` will destroy this session.
* Altering `this.session.cookie` changes the cookie options of this user. Also you can use the cookie options in session the store. Use for example `cookie.maxAge` as the session store's ttl.
* Calling `this.regenerateSession` will destroy any existing session and generate a new, empty one in its place. The new session will have a different ID.


### Options

 * `key`: cookie name defaulting to `koa.sid`
 * `prefix`: session prefix for store, defaulting to `koa:sess:`
 * `ttl`: ttl is for sessionStore's expiration time. it is different with `cookie.maxAge`, default to null(means get ttl from `cookie.maxAge`).
 * `genSid`: default sid was generated by [uid2](https://github.com/coreh/uid2), you can pass a function to replace it
 * `allowEmpty`: allow generation of empty sessions
 * `errorHandler(err, type, ctx)`: `Store.get` and `Store.set` will throw in some situation, use `errorHandle` to handle these errors by yourself. Default will throw.
 * `reconnectTimeout`: When store is disconnected, don't throw `store unavailable` error immediately, wait `reconnectTimeout` to reconnect, default is `10s`.
 * `sessionIdStore`: object with get, set, reset methods for passing session id throw requests.
 * `valid`: valid(ctx, session), valid session value before use it
 * `beforeSave`: beforeSave(ctx, session), hook before save session
 * `store`: session store instance. It can be any Object that has the methods `set`, `get`, `destroy`
 * `cookie`: session cookie settings, defaulting to
  ```js
  {
    httpOnly: true,
    path: '/',
    overwrite: true,
    signed: true,
    maxAge: 24 * 60 * 60 * 1000,
  }
  ```
  For a full list of cookie options see [expressjs/cookies](https://github.com/expressjs/cookies#cookiesset-name--value---options--).
  
  if you set`cookie.maxAge` to `null`, meaning no "expires" parameter is set so the cookie becomes a browser-session cookie. When the user closes the browser the cookie (and session) will be removed.
  
  Notice that `ttl` is different from `cookie.maxAge`, `ttl` set the expire time of sessionStore. So if you set `cookie.maxAge = null`, and `ttl=ms('1d')`, the session will expired after one day, but the cookie will destroy when the user closes the browser.
  And mostly you can just ignore `options.ttl`, `koa-simple-session` will parse `cookie.maxAge` as the tll.

## Hooks

- `valid()`: valid session value before use it
- `beforeSave()`: hook before save sessions

## Session Store

You can use any other store to replace the default MemoryStore, it just needs to follow this api:

* `get(sid)`: get session object by sid
* `set(sid, sess, ttl)`: set session object for sid, with a ttl (in ms)
* `destroy(sid)`: destroy session for sid

the api needs to return a Promise.

And use these events to report the store's status.

* `connect`
* `disconnect`

## License

MIT
