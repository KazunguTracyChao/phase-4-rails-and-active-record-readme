# Architecture

## Overview

![https://bit.ly/2iJuFky](images/puma-general-arch.png)

Puma is a threaded Ruby HTTP application server, processing requests across a TCP or UNIX socket.

Puma processes (there can be one or many) accept connections from the socket via a thread (in the `Reactor` class). The connection, once fully buffered and read, moves in to the `todo` list, where it will be picked up by a free/waiting thread in the threadpool (the `ThreadPool` class).

Puma works in two main modes: cluster and single. In single mode, only one Puma process is booted. In cluster mode, a `master` process is booted, which prepares (and may boot) the application, and then uses the `fork()` system call to create 1 or more `child` processes. These `child` processes all listen to the same socket. The `master` process does not listen to the socket or process requests - its purpose is mostly to manage and listen for UNIX signals and possibly kill or boot `child` processes.

We sometimes call `child` processes (or Puma processes in `single` mode) _workers_, and we sometimes call the threads created by Puma's `ThreadPool` _worker threads_.

## How Requests Work

![https://bit.ly/2zwzhEK](images/puma-connection-flow.png)

* Upon startup, Puma listens on a TCP or UNIX socket.
  * The backlog of this socket is configured (with a default of 1024). This determines the size of the queue for unaccepted connections. Generally, this setting is unimportant and will never be hit in production use. If the backlog is full, the connection will be refused by the operating system.
  * This socket backlog is distinct from the `backlog` of work as reported by `Puma.stats` or the control server. The backlog as reported by Puma is the number of connections in the process' `todo` set waiting for a thread from the `ThreadPool`.
* By default, a single, separate thread (created by the `Reactor` class) is used to read and buffer requests from the socket.
  * When at least one worker thread is available for work, the reactor thread listens to the socket and accepts a request, if one is waiting.
  * The reactor thread waits for the entire HTTP request to be received.
    * The time spent waiting for the HTTP request body to be received is exposed to the Rack app as `env['puma.request_body_wait']` (milliseconds).
  * Once fully buffered and received, the connection is pushed into the "todo" set.
* Worker threads pop work off the "todo" set for processing.
  * The worker thread processes the request via `call`ing the configured Rack application. The Rack application generates the HTTP response.
  * The worker thread writes the response to the connection. Note that while Puma buffers requests via a separate thread, it does not use a separate thread for responses.
  * Once done, the thread become available to process another connection in the "todo" set.

### `queue_requests`

![https://bit.ly/2zxCJ1Z](images/puma-connection-flow-no-reactor.png)

The `queue_requests` option is `true` by default, enabling the separate reactor thread used to buffer requests as described above.

If set to `false`, this buffer will not be used for connections while waiting for the request to arrive.

In this mode, when a connection is accepted, it is added to the "todo" queue immediately, and a worker will synchronously do any waiting necessary to read the HTTP request from the socket.