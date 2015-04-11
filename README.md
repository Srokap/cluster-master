# cluster-master-ext

A module for taking advantage of the built-in `cluster` module in node
v0.8+, enables rolling worker restarts, resizing, repl, events,
configurable timeouts, debug method.

Modified from Isaac's original version `cluster-master` adding:

 - events
 - repl config, help, docs
 - configurable timeouts
 - exports `debug` method for ability to write to all REPLs and console

Note: I had provided these changes as pull requests back to the original author,
but after waiting for 10 months, I will now provide this as an alternative module

Your main `server.js` file uses this module to fire up a cluster of
workers.  Those workers then do the actual server stuff (using socket.io,
express, tako, raw node, whatever; any TCP/TLS/HTTP/HTTPS server would
work.)

This module provides some basic functionality to keep a server running.
As the name implies, it should only be run in the master module, not in
any cluster workers.

```javascript
var clusterMaster = require("cluster-master-ext")

// most basic usage: just specify the worker
// Spins up as many workers as you have CPUs
//
// Note that this is VERY WRONG for a lot of multi-tenanted
// VPS environments where you may have 32 CPUs but only a
// 256MB RSS cap or something. ie. specify the size to
// have what makes sense
clusterMaster("worker.js")

// more advanced usage.  Specify configs.
// in real life, you can only actually call clusterMaster() once.
clusterMaster({ exec: "worker.js" // script to run
              , size: 5 // number of workers
              , env: { SOME: "environment_vars" }
              , args: [ "--deep", "doop" ]
              , silent: true
              , signals: false
              , onMessage: function (msg) {
                  console.error("Message from %s %j"
                               , this.uniqueID
                               , msg)
                }
              })

// methods
clusterMaster.resize(10)

// graceful rolling restart
clusterMaster.restart()

// graceful shutdown
clusterMaster.quit()

// not so graceful shutdown
clusterMaster.quitHard()

// listen to events to additional cleanup or shutdown
clusterMaster.emitter()
  .on('resize', function (clusterSize) { })
  .on('restart', function () { })
  .on('quit', function () { })
  .on('quitHard', function () { });
```

## Install

Use from github or via npm

```bash
npm install cluster-master-ext
```

## Methods

### clusterMaster.resize(n)

Set the cluster size to `n`.  This will disconnect extra nodes and/or
spin up new nodes, as needed.  Done by default on restarts.
Fires `resize` event with new clusterSize just before performing the
resize.

### clusterMaster.restart(cb)

One by one, shut down nodes and spin up new ones.  Callback is called
when finished. Fires `restart` event just before performing restart.

### clusterMaster.quit()

Gracefully shut down the worker nodes and then process.exit(0).
Fires `quit` event just before performing the shutdown.

### clusterMaster.quitHard()

Forcibly shut down the worker nodes and then process.exit(1).
Fires `quitHard` event just before performing hard shut down.

### clusterMaster.emitter()

Retrieve the clusterMaster EventEmitter to be able to listen
to clusterMaster events. This emitter is also returned from
the original clusterMaster() constructor.

### clusterMaster.debug(arg1, arg2, ...)

Arguments passed to debug are formatted with util.format and output
to stdout and any REPL's.

```javascript
clusterMaster.debug('The number one is %s', 1);
```

## Configs

The `exec`, `env`, `argv`, and `silent` configs are passed to the
`cluster.fork()` call directly, and have the same meaning.

* `exec` - The worker script to run
* `env` - Envs to provide to workers
* `argv` - Additional args to pass to workers.
* `silent` - Boolean, default=false.  Do not share stdout/stderr
* `size` - Starting cluster size.  Default = CPU count
* `signals` - Boolean, default=true.  Set up listeners to:
  * `SIGHUP` - restart
  * `SIGINT` - quit (control-c)
  * `SIGABRT` - quitHard
* `onMessage` - Method that gets called when workers send a message to
  the parent.  Called in the context of the worker, so you can reply by
  looking at `this`.
* `stopTimeout` - Time in milliseconds to wait for worker to stop before
  forcefully killing the process during restart or resize, default 5000
  (5 seconds)
* `skepticTimeout` - Time in milliseconds to wait for worker to live
  before shutting previous worker down during restart, default 2000
  (2 seconds)
* `silenceDebug` - if true, then silences the normal console debug messages, default false (output will still continue to repls regardless)
* `aliveEvent` - the cluster event to wait for to consider the child process to be alive, set to `online` for non http workers, default `listening`

* `replHelp` - Array of additional text lines to add to repl `help` command

```javascript
var config = {
  replHelp: [
      'process     - access node.js process',
      '.break      - interrupt current command'
  ]
};
```

* `replContext` - Object of additional properties or functions to add to the REPL context

```javascript
var config = {
  replContext: {
    foo: fooObject  // adds foo to the REPL which exposes the fooObject
  }
};
```

* `repl` - where to have REPL listen, defaults to `env.CLUSTER_MASTER_REPL` || 'cluster-master-socket'
  * if `repl` is null or false - REPL is disabled and will not be started
  * if `repl` is string path - REPL will listen on unix domain socket to this path
  * if `repl` is an integer port - REPL will listen on TCP 0.0.0.0:port
  * if `repl` is an object with `address` and `port`, then REPL will listen on TCP address:PORT

Examples of configuring `repl`

```javascript
var config = { repl: false }                       // disable REPL
var config = { repl: '/tmp/cluster-master-sock' }  // unix domain socket
var config = { repl: 3001 }                        // tcp socket 0.0.0.0:3001
var config = { repl: { address: '127.0.0.1', port: 3002 }}  // tcp 127.0.0.1:3002
```

Note: be careful when using TCP for your REPL since anyone on the
network can connect to your REPL (no security). So either disable
the REPL or use a unix domain socket which requires local access
(or ssh access) to the server.

## REPL

Cluster-master provides a REPL into the master process so you can inspect
the state of your cluster. By default the REPL is accessible by a socket
written to the root of the directory, but you can override it with the
`CLUSTER_MASTER_REPL` environment variable. You can access the REPL with
nc or [socat](http://www.dest-unreach.org/socat/) like so:


```bash
nc -U ./cluster-master-socket

# OR

socat ./cluster-master-socket stdin
```

The REPL provides you with access to these objects or functions:

* `help`        - display these commands
* `repl`        - access the REPL
* `resize(n)`   - resize the cluster to `n` workers
* `restart(cb)` - gracefully restart workers, cb is optional
* `stop()`      - gracefully stop workers and master
* `kill()`      - forcefully kill workers and master
* `cluster`     - node.js cluster module
* `size`        - current cluster size
* `connections` - number of REPL connections to master
* `workers`     - current workers
* `select(fld)` - map of id to `field` (from workers)
* `pids`        - map of id to pids
* `ages`        - map of id to worker ages
* `states`      - map of id to worker states
* `debug(a1)`   - output `a1` to stdout and all REPLs
* `sock`        - this REPL socket'
* `.exit`       - close this connection to the REPL

## Events

clusterMaster emits events on clusterMaster.emitter() when its methods
are called which allows you to respond and do additional cleanup right
before the action is carried out.

* `debug` - fired when debug() is called to output messages, listener ex: `fn(msg, args, ...)`
* `disconnect` - fired before worker is to be disconnected, listener ex: `fn(worker)`
* `resize` - fired on clusterMaster.resize(n), listener ex: `fn(clusterSize)`
* `restart` - fired on clusterMaster.restart(), listener ex: `fn(oldWorkers)`
  `restartComplete` - fired when restart is completed
* `quit` - fired on clusterMaster.quit()
* `quitHard` - fired on clusterMaster.quitHard()

## LICENSE

BSD

## Authors and Contributors

 - Isaac Z. Schlueter, author of the original project `cluster-master`
 - Jeff Barczewski
 - Sean McCullough
