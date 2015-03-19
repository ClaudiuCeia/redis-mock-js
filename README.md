# redis-mock-js
An in-memory redis-compatible implementation written in pure
javascript.

### Status
[![Build Status](https://travis-ci.org/wilkenstein/redis-mock-js.svg?branch=master)](https://travis-ci.org/wilkenstein/redis-mock-js)

## Installation

### npm

````bash
$ npm install redis-js
````

### Browser

There are minified files available on github. These files currently
represent the latest version. To use in the browser, add the following
script tag:

````html
<script src="https://github.com/wilkenstein/redis-mock-min.js">
````

## Usage

### node.js/io.js

#### Basic Usage

The below code demonstrates a toy example of using the client in
node.js/io.js. The toy example sets a key, gets it, then deletes it.

````javascript
var redisClient = require('redis-js');
var async = require('async');
var Q = require('q');

// Return blocking style.
var reply = redisClient.set('key', 'value');
if (reply === 'OK') {
    var value = redisClient.get('key');
    if (value === 'value') {
        redisClient.del('key');
    }
}

// Callback style (pyramid).
function setGetDel(callback) {
    redisClient.set('key', 'value', function (err, reply) {
        if (err || reply !== 'OK') {
            return callback(err, reply);
        }
        redisClient.get('key', function (err2, reply2) {
            if (err2 || reply2 !== 'value') {
                return callback(err2, reply2);
            }
            redisClient.del('key', function (err3, reply3) {
                if (err3) {
                    return callback(err3);
                }
                return callback(err3, reply3);
            });
        });
    });
}
setGetDel(function (err) { if (err) throw err; });

// Async style.
function setGetDelAsync(callback) {
    async.waterfall([
        function (cb) {
            redisClient.set('key', 'value', cb);
        },
        function (reply, cb) {
            if (reply !== 'OK') {
                return cb(new Error('reply not OK'));
            }
            redisClient.get('key', cb);
        },
        function (reply, cb) {
            if (reply !== 'value') {
                return cb(new Error('reply !== value'));
            }
            redisClient.del('key', cb);
        }
    ], function (err, reply) {
        callback(err, reply);
    });
}
setGetDelAsync(function (err) { if (err) throw err; });
    
// Promise style with q.
function setGetDelQ() {
    return Q
        .nfcall(redisClient.set, 'key', 'value')
        .then(function (reply) {
            if (reply !== 'OK') {
                throw new Error('reply not OK');
            }
            return Q.nfcall(redisClient.get, 'key');
        }, function (err) {
            throw err;
        })
        .then(function (reply) {
            if (reply !== 'value') {
                throw new Error('reply !== value');
            }
            return Q.nfcall(redisClient.del, 'key');
        }, function (err) {
            throw err;
        });
}
setGetDelQ()
    .fail(function (err) { throw err; })
    .done();
````

### Browser

#### Basic Usage

The redis-mock object is exported onto the global `window` object as
`window.redismock`. Example toy usage:

````javascript
(function () {

    var redis = this.redismock;

    redis.set('key', 'value');
    console.log(redis.get('key')); // Logs 'value' to the console.

})();
````

### toPromiseStyle

A convenience function exists on the redismock object to turn all the
redis commands into a Promise style. This function takes a Factory
Function that creates deferred objects, e.g., `Q.defer` or
`jQuery.Deferred`. Example usage:

````javascript
var Q = require('q');
var redis = require('redis-js').toPromiseStyle(Q.defer);

function setGetAsPromise() {
    return redis
        .set('key', 'value')
        .then(function () {
            return redis.get('key');
        })
        .then(function (value) {
            console.log(value);
        })
        .fail(function (err) {
            // Should not be exercised.
            console.log(err);
        })
        .done();
}
setGetAsPromise() // On console, 'value' should be logged.
````

### toNodeRedis

It seems silly to write a bunch of unit tests using redismock, then
create a separate path for integration testing against a real redis
server, no? That's where `toNodeRedis` comes in!

`toNodeRedis` allows you to convert the redismock into an actual redis
client using the [node_redis](https://github.com/mranney/node_redis) package. This function will do nothing in
a non-CommonJS environment, and throw an error if the `redis` package
is not installed. To facilitate automatic switching, the environment
variable `REDIS_JS_TO_NODE_REDIS` can be set to `1`. If you need to
connect to a specific redis server with specific options, pass the same
arguments to `toNodeRedis` as you would to `redis.createClient`. When
automatically switching with environment variables, use
`REDIS_JS_NODE_REDIS_PORT`, `REDIS_JS_NODE_REDIS_HOST`, and
`REDIS_JS_NODE_REDIS_OPTIONS`. These environment variables will be
passed onto `redis.createClient`. `REDIS_JS_NODE_REDIS_OPTIONS` must
be a valid JSON string, or the script will throw an error.

`toPromiseStyle` is still available on the newly-formed redismock ->
node redis client, and is compatible with node redis.

Example usage:

````javascript
var Q = require('q');
var client = require('redis-js').toNodeRedis().toPromiseStyle(Q.defer);

client
    .set('k', 'v')
    .then(function (reply) {
        console.log(reply);
        return redis.get('k');
    })
    .then(function (reply) {
        console.log(reply);
        return redis.del('k');
    })
    .then(function (reply) {
        console.log(reply);
    })
    .fail(function (err) {
        throw err;
    })
    .done();
    
// Result of this script should be console logs of
//   'OK'
//   'v'
//   1
````

## Supported Commands

The goal is to have one-to-one feature parity with all redis commands, so that this implementation can simply be dropped into an existing redis-backed codebase. Redis has a lot of commands! Some of them are easy, and some are quite complex.

Version 0.1.0 should have support for almost all redis commands, minus the hyperloglog commands and the key-movement commands, such as migrate.

To find out what commands a particular version of redis-js supports, run the following commands:

````bash
$ npm run implemented
$ npm run unimplemented
````

Theses two commands will print out all the implemented and unimplemented commands, respectively.

NOTE: version 0.0.5 will have full support for implemented and
unimplemented. Versions <= 0.0.4 do <i>not</i> support these commands
and will be deprecated at some point.

## Contributing

All contributions welcome! Issues or Pull Requests. For PRs, `npm test`, `npm
run test-phantomjs`, and `npm run build-min && npm run test-min` must
all succeed and test the new code before the PR will be considered.

## Testing

This project uses [jshint](http://jshint.com),
[karma](http://karma-runner.github.io/0.12/index.html) +
[mocha](http://mochajs.org) + [chai](http://chaijs.com), and [plato](https://github.com/es-analysis/plato).

To run the full test suite from source, issue the standard npm command:

````bash
$ npm test
````

This will run jshint against all the source files, run the full mocha suite with a coverage report, and finally run a complexity report against all the source files. Currently, the only source file is redis-mock.js.

To run the mocha tests within a particular browser using karma, issue the following npm commands:

````bash
$ npm run test-phantomjs
$ npm run test-firefox
$ npm run test-chrome
$ npm run test-safari
$ npm run test-opera
$ npm run test-ie
````

These tests use the various karma-*-launcher projects to launch the browsers. If the browser is not installed on the system, the corresponding test command will print an error and hang.

test-phantomjs is the only test command that does not rely on external
dependencies. test-phantomjs will install the latest compatible
version of phantomjs in a tmp directory, and use that binary to run
the tests.

Creating a test is as easy as adding a new test file in
`test/mocha/`. In order for the test to run across different
JavaScript engines, a somewhat specific style is required:

````javascript
(function() {
    var redismock = typeof require === 'function' ? require('../../redis-mock.js') : window.redismock;
    if (typeof chai === 'undefined') {
        var chai = typeof require === 'function' ? require('chai') : window.chai;
    }
    chai.config.includeStack = true;
    var expect = chai.expect;

    /* Put describe & it blocks here */
    
}).call(this);
````

## Roadmap

* 0.1.0
  - Support for most redis commands.
  - Mocha tests for all supported commands.
  - Browser testing and compatibility determination.
* 1.0.0
  - Support for different versions of mock redis that mimic different
    redis versions.
  - Support for multiple mock redis instances.
  - Support for migrating data between mock redis instances.
  - Server support.
* 2.0.0
  - Support for migrating data from a mock redis instance to a real
    redis instance.
  - Support for persisting a mock redis instance.
  - HyperLogLog support.

## Versions
* 0.0.10-2
  - Sorted Set time improvements at the cost of memory.
* 0.0.10-1
  - Sorted Set time improvements.
  - Load tests for sorted sets.
* 0.0.10
  - All sorted set commands implemented and tested.
* 0.0.9-5
  - Fix for multi commands to return errors how redis returns errors on exec.
* 0.0.9-4
  - Implemented all sorted set commands, but not yet tested.
* 0.0.9-3
  - Set time & space improvements.
* 0.0.9-2
  - Bug fix for lpush. In redis, lpush(k, v1, v2) = k: [v2, v1].
* 0.0.9-1
  - Bug fix on converting redismock to node redis using process env vars.
* 0.0.9
  - Add support for converting redismock to node redis.
* 0.0.8-1
  - Set diff, inter, and union performance improvements.
  - 1st implementation of load testing.
* 0.0.8
  - Implement all set commands, including sscan.
  - Unit test all set commands.
* 0.0.7-1
  - lrange fix for negative indices.
* 0.0.7
  - All string commands implemented.
  - Better unit tests.
  - Automatically generate and test a minified redis-mock.
* 0.0.6
  - Implement more commands.
  - Implement more unit tests.
* 0.0.5
  - Full browser testing capabilities.
  - All redis commands "un"-implemented.
  - Added way to see implemented vs unimplemented commands.
  - More unit tests and more implementations.
* 0.0.4
  - DEPRECATED
  - Enhanced hash support.
  - Unit tests for hash commands.
* 0.0.3
  - DEPRECATED
  - Set, sorted set, and hash support.
  - Unit tests for implemented list, set, and sorted set commands.
  - Bug squashes.
* 0.0.2
  - DEPRECATED
  - Transaction support that works.
  - Bug squashes.
  - Enhanced commands.
* 0.0.1
  - DEPRECATED
  - Initial implementation.
  - Numerous bugs.
  - Incomplete commands.
