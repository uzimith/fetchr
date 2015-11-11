# Fetchr 

[![npm version](https://badge.fury.io/js/fetchr.svg)](http://badge.fury.io/js/fetchr)
[![Build Status](https://travis-ci.org/yahoo/fetchr.svg?branch=master)](https://travis-ci.org/yahoo/fetchr)
[![Dependency Status](https://david-dm.org/yahoo/fetchr.svg)](https://david-dm.org/yahoo/fetchr)
[![devDependency Status](https://david-dm.org/yahoo/fetchr/dev-status.svg)](https://david-dm.org/yahoo/fetchr#info=devDependencies)
[![Coverage Status](https://coveralls.io/repos/yahoo/fetchr/badge.png?branch=master)](https://coveralls.io/r/yahoo/fetchr?branch=master) 

Universal data access layer for web applications.

Typically on the server, you call your API or database directly to fetch some data. However, on the client, you cannot always call your services in the same way (i.e, cross domain policies). Instead, XHR requests need to be made to the server which get forwarded to your service.

Having to write code differently for both environments is duplicative and error prone. Fetchr provides an abstraction layer over your data service calls so that you can fetch data using the same API on the server and client side. 

## Install

```
npm install fetchr --save
```

## Setup

Follow the steps below to setup Fetchr properly. This assumes you are using the [Express](https://www.npmjs.com/package/express) framework.

### 1. Configure Server

On the server side, add the Fetchr middleware into your express app at a custom API endpoint.

Fetchr middleware expects that you're using the [`body-parser`](https://github.com/expressjs/body-parser) middleware (or an alternative middleware that populates `req.body`) before you use Fetchr middleware.

```js
var express = require('express');
var Fetcher = require('fetchr');
var bodyParser = require('body-parser');
var app = express();

// you need to use body-parser middleware before fetcher middleware
app.use(bodyParser.json());

app.use('/myCustomAPIEndpoint', Fetcher.middleware());
```

### 2. Configure Client

On the client side, it is necessary for the `xhrPath` option to match the path where the middleware was mounted in the previous step

`xhrPath` is an optional config property that allows you to customize the endpoint to your services, defaults to `/api`.

```js
var Fetcher = require('fetchr');
var fetcher = new Fetcher({
    xhrPath: '/myCustomAPIEndpoint'
});
```

### 3. Register data services

You will need to register any data services that you wish to use in your application.
The interface for your service will be an object that must define a `name` property and at least one [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operation.
The `name` propety will be used when you call one of the CRUD operations.

```js
// app.js
var Fetcher = require('fetchr');
var myDataService = require('./dataService');
Fetcher.registerService(myDataService);
```

```js
// dataService.js
module.exports = {
    // name is required
    name: 'data_service',
    // at least one of the CRUD methods is required
    read: function(req, resource, params, config, callback) {
      //...
    },
    // other methods
    // create: function(req, resource, params, body, config, callback) {},
    // update: function(req, resource, params, body, config, callback) {},
    // delete: function(req, resource, params, config, callback) {}
}
```

### 4. Instantiating the Fetchr Class

Data services might need access to each individual request, for example, to get the current logged in user's session.
For this reason, Fetcher will have to be instantiated once per request.

On the serverside, this requires fetcher to be instantiated per request, in express middleware.
On the clientside, this only needs to happen on page load.


```js
// app.js - server
var express = require('express');
var Fetcher = require('fetchr');
var app = express();
var myDataService = require('./dataService');

// register the service
Fetcher.registerService(myDataService);

// register the middleware
app.use('/myCustomAPIEndpoint', Fetcher.middleware());

app.use(function(req, res, next) {
    // instantiated fetcher with access to req object
    var fetcher = new Fetcher({
        xhrPath: '/myCustomAPIEndpoint', // xhrPath will be ignored on the serverside fetcher instantiation
        req: req
    });

    // perform read call to get data
    fetcher
        .read('data_service')
        .params({id: ###})
        .end(function (err, data, meta) {
        // handle err and/or data returned from data fetcher in this callback
        });
});
```

```js
// app.js - client
var Fetcher = require('fetchr');
var fetcher = new Fetcher({
    xhrPath: '/myCustomAPIEndpoint' // xhrPath is REQUIRED on the clientside fetcher instantiation
});
fetcher
    .read('data_api_fetcher')
    .params({id: ###})
    .end(function (err, data, meta) {
    // handle err and/or data returned from data fetcher in this callback
    });
    
// for create you can use the body() method to pass data
fetcher
    .create('data_api_create')
    .body({"some":"data");
    .end(function (err, data, meta) {
    // handle err and/or data returned from data fetcher in this callback
    });
```



## Usage Examples

See the [simple example](https://github.com/yahoo/fetchr/tree/master/examples/simple).

## Service Metadata

Service calls on the client transparently become xhr requests.
It is a good idea to set cache headers on common xhr calls.
You can do so by providing a third parameter in your service's callback.
If you want to look at what headers were set by the service you just called,
simply inspect the third parameter in the callback.

Note: If you're using promises, the metadata will be available on the `meta`
property of the resolved value.

```js
// dataService.js
module.exports = {
    name: 'data_service',
    read: function(req, resource, params, config, callback) {
        // business logic
        var data = 'response';
        var meta = {
            headers: {
                'cache-control': 'public, max-age=3600'
            },
            statusCode: 200 // You can even provide a custom statusCode for the xhr response
        };
        callback(null, data, meta);
    }
}
```

```js
fetcher
    .read('data_service')
    .params({id: ###})
    .end(function (err, data, meta) {
        // data will be 'response'
        // meta will have the header and statusCode from above
    });
```

There is a convenience method called `fetcher.getServiceMeta` on the fetchr instance.
This method will return the metadata for all the calls that have happened so far
in an array format.
In the server, this will include all service calls for the current request.
In the client, this will include all service calls for the current session.


## Updating Configuration

Usually you instantiate fetcher with some default options for the entire browser session,
but there might be cases where you want to update these options later in the same session.

You can do that with the `updateOptions` method:

```js
// Start
var fetcher = new Fetcher({
    xhrPath: '/myCustomAPIEndpoint',
    xhrTimeout: 2000
});

// Later, you may want to update the xhrTimeout
fetcher.updateOptions({
    xhrTimeout: 4000
});
```


## Error Handling

When an error occurs in your Fetchr CRUD method, you should return an error object to the callback. The error object should contain a `statusCode` (default 400) and `output` property that contains valid `JSON` which will be serialized to the client.

```js
module.exports = {
    name: 'FooService',
    read: function create(req, resource, params, configs, callback) {
        var err = new Error('it failed');
        err.statusCode = 404;
        err.output = { message: "Not found", more: "meta data" };
        return callback(err);
    }
};
```

## XHR Timeouts

`xhrTimeout` is an optional config property that allows you to set timeout (in ms) for all clientside requests, defaults to `3000`.
On the clientside, xhrPath and xhrTimeout will be used for XHR requests.
On the serverside, xhrPath and xhrTimeout are not needed and are ignored.

```js
var Fetcher = require('fetchr');
var fetcher = new Fetcher({
    xhrPath: '/myCustomAPIEndpoint',
    xhrTimeout: 4000
});
```

If you have an individual request that you need to ensure has a specific timeout you can do that via the `timeout` option in `clientConfig`:

```js
fetcher
    .read('someData')
    .params({id: ###})
    .clientConfig({timeout: 5000}) // wait 5 seconds for this request before timing it out
    .end(function (err, data, meta) {
    // handle err and/or data returned from data fetcher in this callback
    });
```

## CORS Support

Fetchr provides [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) support by allowing you to pass the full origin host into `corsPath` option.

For example:

```js
var Fetcher = require('fetchr');
var fetcher = new Fetcher({
    corsPath: 'http://www.foo.com',
    xhrPath: '/fooProxy'
});
fetcher
    .read('service')
    .params({ foo: 1 })
    .clientConfig({ cors: true })
    .end(callbackFn);
```

Additionally, you can also customize how the GET URL is constructed by passing in the `constructGetUri` property when you execute your `read` call:

```js
var qs = require('qs');
function customConstructGetUri(uri, resource, params, config) {
    // this refers to the Fetcher object itself that this function is invoked with.
    if (config.cors) {
        return uri + '/' + resource + '?' + qs.stringify(this.context);
    }
    // Return `falsy` value will result in `fetcher` using its internal path construction instead.
}

var Fetcher = require('fetchr');
var fetcher = new Fetcher({
    corsPath: 'http://www.foo.com',
    xhrPath: '/fooProxy'
});
fetcher
    .read('service')
    .params({ foo: 1 })
    .clientConfig({
        cors: true,
        constructGetUri: customConstructGetUri
    })
    .end(callbackFn);
```


## CSRF Protection

You can protect your XHR paths from CSRF attacks by adding a middleware in front of the fetchr middleware:

`app.use('/myCustomAPIEndpoint', csrf(), Fetcher.middleware());`

You could use https://github.com/expressjs/csurf for this as an example.

Next you need to make sure that the CSRF token is being sent with our XHR requests so that they can be validated. To do this, pass the token in as a key in the `options.context` object on the client:

```js
var fetcher = new Fetcher({
    xhrPath: '/myCustomAPIEndpoint', //xhrPath is REQUIRED on the clientside fetcher instantiation
    context: { // These context values are persisted with XHR calls as query params
        _csrf: 'Ax89D94j'
    }
});
```

This `_csrf` will be sent in all XHR requests as a query parameter so that it can be validated on the server.

## Service Call Config

When calling a Fetcher service you can pass an optional config object.

When this call is made from the client, the config object is used to define XHR request options and can be used to override default options:

```js
//app.js - client
var config = {
    timeout: 6000, // Timeout (in ms) for each request
    unsafeAllowRetry: false // for POST requests, whether to allow retrying this post
};

fetcher
    .read('service')
    .params({ id: 1 })
    .clientConfig(config)
    .end(callbackFn);
```

For requests from the server, the config object is simply passed into the service being called.

## Context Variables

By Default, fetchr appends all context values to the xhr url as query params. `contextPicker` allows you to greater control over which context variables get sent as query params depending on the xhr method (`GET` or `POST`). This is useful when you want to limit the number of variables in a `GET` url in order not to accidentally [cache bust](http://webassets.readthedocs.org/en/latest/expiring.html).

`contextPicker` follows the same format as the `predicate` parameter in [`lodash/object/pick`](https://lodash.com/docs#pick) with three arguments: `(value, key, object)`.

```js
var fetcher = new Fetcher({
    context: { // These context values are persisted with XHR calls as query params
        _csrf: 'Ax89D94j',
        device: 'desktop'
    },
    contextPicker: {
        GET: function (value, key, object) {
            // for example, if you don't enable CSRF protection for GET, you are able to ignore it with the url
            if (key === '_csrf') {
                return false;
            }
            rerurn true;
        }
        // for other method e.g., POST, if you don't define the picker, it will pick the entire context object
    }
});

var fetcher = new Fetcher({
    context: { // These context values are persisted with XHR calls as query params
        _csrf: 'Ax89D94j',
        device: 'desktop'
    },
    contextPicker: {
        GET: ['device'] // predicate can be an array of strings
    }
});
```

## API

- [Fetchr](https://github.com/yahoo/fetchr/blob/master/docs/fetchr.md)

## License

This software is free to use under the Yahoo! Inc. BSD license.
See the [LICENSE file][] for license text and copyright information.

[LICENSE file]: https://github.com/yahoo/fetchr/blob/master/LICENSE.md
