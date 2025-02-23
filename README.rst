redis-py
========

The Python interface to the Redis key-value store.

.. image:: https://secure.travis-ci.org/andymccurdy/redis-py.svg?branch=master
        :target: https://travis-ci.org/andymccurdy/redis-py
.. image:: https://readthedocs.org/projects/redis-py/badge/?version=latest&style=flat
        :target: https://redis-py.readthedocs.io/en/latest/
.. image:: https://badge.fury.io/py/redis.svg
        :target: https://pypi.org/project/redis/
.. image:: https://codecov.io/gh/andymccurdy/redis-py/branch/master/graph/badge.svg
  :target: https://codecov.io/gh/andymccurdy/redis-py

Installation
------------

redis-py requires a running Redis server. See `Redis's quickstart
<https://redis.io/topics/quickstart>`_ for installation instructions.

redis-py can be installed using `pip` similar to other Python packages. Do not use `sudo`
with `pip`. It is usually good to work in a
`virtualenv <https://virtualenv.pypa.io/en/latest/>`_ or
`venv <https://docs.python.org/3/library/venv.html>`_ to avoid conflicts with other package
managers and Python projects. For a quick introduction see
`Python Virtual Environments in Five Minutes <https://bit.ly/py-env>`_.

To install redis-py, simply:

.. code-block:: bash

    $ pip install redis

or from source:

.. code-block:: bash

    $ python setup.py install


Getting Started
---------------

.. code-block:: pycon

    >>> import redis
    >>> r = redis.Redis(host='localhost', port=6379, db=0)
    >>> r.set('foo', 'bar')
    True
    >>> r.get('foo')
    'bar'

By default, all responses are returned as `bytes` in Python 3 and `str` in
Python 2. The user is responsible for decoding to Python 3 strings or Python 2
unicode objects.

If **all** string responses from a client should be decoded, the user can
specify `decode_responses=True` to `Redis.__init__`. In this case, any
Redis command that returns a string type will be decoded with the `encoding`
specified.


Upgrading from redis-py 2.X to 3.0
----------------------------------

redis-py 3.0 introduces many new features but required a number of backwards
incompatible changes to be made in the process. This section attempts to
provide an upgrade path for users migrating from 2.X to 3.0.


Python Version Support
^^^^^^^^^^^^^^^^^^^^^^

redis-py 3.0 now supports Python 2.7 and Python 3.4+. Python 2.6 and 3.3
support has been dropped.


Client Classes: Redis and StrictRedis
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

redis-py 3.0 drops support for the legacy "Redis" client class. "StrictRedis"
has been renamed to "Redis" and an alias named "StrictRedis" is provided so
that users previously using "StrictRedis" can continue to run unchanged.

The 2.X "Redis" class provided alternative implementations of a few commands.
This confused users (rightfully so) and caused a number of support issues. To
make things easier going forward, it was decided to drop support for these
alternate implementations and instead focus on a single client class.

2.X users that are already using StrictRedis don't have to change the class
name. StrictRedis will continue to work for the forseeable future.

2.X users that are using the Redis class will have to make changes if they
use any of the following commands:

* SETEX: The argument order has changed. The new order is (name, time, value).
* LREM: The argument order has changed. The new order is (name, num, value).
* TTL and PTTL: The return value is now always an int and matches the
  official Redis command (>0 indicates the timeout, -1 indicates that the key
  exists but that it has no expire time set, -2 indicates that the key does
  not exist)


SSL Connections
^^^^^^^^^^^^^^^

redis-py 3.0 changes the default value of the `ssl_cert_reqs` option from
`None` to `'required'`. See
`Issue 1016 <https://github.com/andymccurdy/redis-py/issues/1016>`_. This
change enforces hostname validation when accepting a cert from a remote SSL
terminator. If the terminator doesn't properly set the hostname on the cert
this will cause redis-py 3.0 to raise a ConnectionError.

This check can be disabled by setting `ssl_cert_reqs` to `None`. Note that
doing so removes the security check. Do so at your own risk.

It has been reported that SSL certs received from AWS ElastiCache do not have
proper hostnames and turning off hostname verification is currently required.


MSET, MSETNX and ZADD
^^^^^^^^^^^^^^^^^^^^^

These commands all accept a mapping of key/value pairs. In redis-py 2.X
this mapping could be specified as ``*args`` or as ``**kwargs``. Both of these
styles caused issues when Redis introduced optional flags to ZADD. Relying on
``*args`` caused issues with the optional argument order, especially in Python
2.7. Relying on ``**kwargs`` caused potential collision issues of user keys with
the argument names in the method signature.

To resolve this, redis-py 3.0 has changed these three commands to all accept
a single positional argument named mapping that is expected to be a dict. For
MSET and MSETNX, the dict is a mapping of key-names -> values. For ZADD, the
dict is a mapping of element-names -> score.

MSET, MSETNX and ZADD now look like:

.. code-block:: pycon

    def mset(self, mapping):
    def msetnx(self, mapping):
    def zadd(self, name, mapping, nx=False, xx=False, ch=False, incr=False):

All 2.X users that use these commands must modify their code to supply
keys and values as a dict to these commands.


ZINCRBY
^^^^^^^

redis-py 2.X accidentily modified the argument order of ZINCRBY, swapping the
order of value and amount. ZINCRBY now looks like:

.. code-block:: pycon

    def zincrby(self, name, amount, value):

All 2.X users that rely on ZINCRBY must swap the order of amount and value
for the command to continue to work as intended.


Encoding of User Input
^^^^^^^^^^^^^^^^^^^^^^

redis-py 3.0 only accepts user data as bytes, strings or numbers (ints, longs
and floats). Attempting to specify a key or a value as any other type will
raise a DataError exception.

redis-py 2.X attempted to coerce any type of input into a string. While
occasionally convenient, this caused all sorts of hidden errors when users
passed boolean values (which were coerced to 'True' or 'False'), a None
value (which was coerced to 'None') or other values, such as user defined
types.

All 2.X users should make sure that the keys and values they pass into
redis-py are either bytes, strings or numbers.


Locks
^^^^^

redis-py 3.0 drops support for the pipeline-based Lock and now only supports
the Lua-based lock. In doing so, LuaLock has been renamed to Lock. This also
means that redis-py Lock objects require Redis server 2.6 or greater.

2.X users that were explicitly referring to "LuaLock" will have to now refer
to "Lock" instead.


Locks as Context Managers
^^^^^^^^^^^^^^^^^^^^^^^^^

redis-py 3.0 now raises a LockError when using a lock as a context manager and
the lock cannot be acquired within the specified timeout. This is more of a
bug fix than a backwards incompatible change. However, given an error is now
raised where none was before, this might alarm some users.

2.X users should make sure they're wrapping their lock code in a try/catch
like this:

.. code-block:: pycon

    try:
        with r.lock('my-lock-key', blocking_timeout=5) as lock:
            # code you want executed only after the lock has been acquired
    except LockError:
        # the lock wasn't acquired


API Reference
-------------

The `official Redis command documentation <https://redis.io/commands>`_ does a
great job of explaining each command in detail. redis-py attempts to adhere
to the official command syntax. There are a few exceptions:

* **SELECT**: Not implemented. See the explanation in the Thread Safety section
  below.
* **DEL**: 'del' is a reserved keyword in the Python syntax. Therefore redis-py
  uses 'delete' instead.
* **MULTI/EXEC**: These are implemented as part of the Pipeline class. The
  pipeline is wrapped with the MULTI and EXEC statements by default when it
  is executed, which can be disabled by specifying transaction=False.
  See more about Pipelines below.
* **SUBSCRIBE/LISTEN**: Similar to pipelines, PubSub is implemented as a separate
  class as it places the underlying connection in a state where it can't
  execute non-pubsub commands. Calling the pubsub method from the Redis client
  will return a PubSub instance where you can subscribe to channels and listen
  for messages. You can only call PUBLISH from the Redis client (see
  `this comment on issue #151
  <https://github.com/andymccurdy/redis-py/issues/151#issuecomment-1545015>`_
  for details).
* **SCAN/SSCAN/HSCAN/ZSCAN**: The \*SCAN commands are implemented as they
  exist in the Redis documentation. In addition, each command has an equivilant
  iterator method. These are purely for convenience so the user doesn't have
  to keep track of the cursor while iterating. Use the
  scan_iter/sscan_iter/hscan_iter/zscan_iter methods for this behavior.


More Detail
-----------

Connection Pools
^^^^^^^^^^^^^^^^

Behind the scenes, redis-py uses a connection pool to manage connections to
a Redis server. By default, each Redis instance you create will in turn create
its own connection pool. You can override this behavior and use an existing
connection pool by passing an already created connection pool instance to the
connection_pool argument of the Redis class. You may choose to do this in order
to implement client side sharding or have finer grain control of how
connections are managed.

.. code-block:: pycon

    >>> pool = redis.ConnectionPool(host='localhost', port=6379, db=0)
    >>> r = redis.Redis(connection_pool=pool)

Connections
^^^^^^^^^^^

ConnectionPools manage a set of Connection instances. redis-py ships with two
types of Connections. The default, Connection, is a normal TCP socket based
connection. The UnixDomainSocketConnection allows for clients running on the
same device as the server to connect via a unix domain socket. To use a
UnixDomainSocketConnection connection, simply pass the unix_socket_path
argument, which is a string to the unix domain socket file. Additionally, make
sure the unixsocket parameter is defined in your redis.conf file. It's
commented out by default.

.. code-block:: pycon

    >>> r = redis.Redis(unix_socket_path='/tmp/redis.sock')

You can create your own Connection subclasses as well. This may be useful if
you want to control the socket behavior within an async framework. To
instantiate a client class using your own connection, you need to create
a connection pool, passing your class to the connection_class argument.
Other keyword parameters you pass to the pool will be passed to the class
specified during initialization.

.. code-block:: pycon

    >>> pool = redis.ConnectionPool(connection_class=YourConnectionClass,
                                    your_arg='...', ...)

Parsers
^^^^^^^

Parser classes provide a way to control how responses from the Redis server
are parsed. redis-py ships with two parser classes, the PythonParser and the
HiredisParser. By default, redis-py will attempt to use the HiredisParser if
you have the hiredis module installed and will fallback to the PythonParser
otherwise.

Hiredis is a C library maintained by the core Redis team. Pieter Noordhuis was
kind enough to create Python bindings. Using Hiredis can provide up to a
10x speed improvement in parsing responses from the Redis server. The
performance increase is most noticeable when retrieving many pieces of data,
such as from LRANGE or SMEMBERS operations.

Hiredis is available on PyPI, and can be installed via pip just like redis-py.

.. code-block:: bash

    $ pip install hiredis

Response Callbacks
^^^^^^^^^^^^^^^^^^

The client class uses a set of callbacks to cast Redis responses to the
appropriate Python type. There are a number of these callbacks defined on
the Redis client class in a dictionary called RESPONSE_CALLBACKS.

Custom callbacks can be added on a per-instance basis using the
set_response_callback method. This method accepts two arguments: a command
name and the callback. Callbacks added in this manner are only valid on the
instance the callback is added to. If you want to define or override a callback
globally, you should make a subclass of the Redis client and add your callback
to its RESPONSE_CALLBACKS class dictionary.

Response callbacks take at least one parameter: the response from the Redis
server. Keyword arguments may also be accepted in order to further control
how to interpret the response. These keyword arguments are specified during the
command's call to execute_command. The ZRANGE implementation demonstrates the
use of response callback keyword arguments with its "withscores" argument.

Thread Safety
^^^^^^^^^^^^^

Redis client instances can safely be shared between threads. Internally,
connection instances are only retrieved from the connection pool during
command execution, and returned to the pool directly after. Command execution
never modifies state on the client instance.

However, there is one caveat: the Redis SELECT command. The SELECT command
allows you to switch the database currently in use by the connection. That
database remains selected until another is selected or until the connection is
closed. This creates an issue in that connections could be returned to the pool
that are connected to a different database.

As a result, redis-py does not implement the SELECT command on client
instances. If you use multiple Redis databases within the same application, you
should create a separate client instance (and possibly a separate connection
pool) for each database.

It is not safe to pass PubSub or Pipeline objects between threads.

Pipelines
^^^^^^^^^

Pipelines are a subclass of the base Redis class that provide support for
buffering multiple commands to the server in a single request. They can be used
to dramatically increase the performance of groups of commands by reducing the
number of back-and-forth TCP packets between the client and server.

Pipelines are quite simple to use:

.. code-block:: pycon

    >>> r = redis.Redis(...)
    >>> r.set('bing', 'baz')
    >>> # Use the pipeline() method to create a pipeline instance
    >>> pipe = r.pipeline()
    >>> # The following SET commands are buffered
    >>> pipe.set('foo', 'bar')
    >>> pipe.get('bing')
    >>> # the EXECUTE call sends all buffered commands to the server, returning
    >>> # a list of responses, one for each command.
    >>> pipe.execute()
    [True, 'baz']

For ease of use, all commands being buffered into the pipeline return the
pipeline object itself. Therefore calls can be chained like:

.. code-block:: pycon

    >>> pipe.set('foo', 'bar').sadd('faz', 'baz').incr('auto_number').execute()
    [True, True, 6]

In addition, pipelines can also ensure the buffered commands are executed
atomically as a group. This happens by default. If you want to disable the
atomic nature of a pipeline but still want to buffer commands, you can turn
off transactions.

.. code-block:: pycon

    >>> pipe = r.pipeline(transaction=False)

A common issue occurs when requiring atomic transactions but needing to
retrieve values in Redis prior for use within the transaction. For instance,
let's assume that the INCR command didn't exist and we need to build an atomic
version of INCR in Python.

The completely naive implementation could GET the value, increment it in
Python, and SET the new value back. However, this is not atomic because
multiple clients could be doing this at the same time, each getting the same
value from GET.

Enter the WATCH command. WATCH provides the ability to monitor one or more keys
prior to starting a transaction. If any of those keys change prior the
execution of that transaction, the entire transaction will be canceled and a
WatchError will be raised. To implement our own client-side INCR command, we
could do something like this:

.. code-block:: pycon

    >>> with r.pipeline() as pipe:
    ...     while True:
    ...         try:
    ...             # put a WATCH on the key that holds our sequence value
    ...             pipe.watch('OUR-SEQUENCE-KEY')
    ...             # after WATCHing, the pipeline is put into immediate execution
    ...             # mode until we tell it to start buffering commands again.
    ...             # this allows us to get the current value of our sequence
    ...             current_value = pipe.get('OUR-SEQUENCE-KEY')
    ...             next_value = int(current_value) + 1
    ...             # now we can put the pipeline back into buffered mode with MULTI
    ...             pipe.multi()
    ...             pipe.set('OUR-SEQUENCE-KEY', next_value)
    ...             # and finally, execute the pipeline (the set command)
    ...             pipe.execute()
    ...             # if a WatchError wasn't raised during execution, everything
    ...             # we just did happened atomically.
    ...             break
    ...        except WatchError:
    ...             # another client must have changed 'OUR-SEQUENCE-KEY' between
    ...             # the time we started WATCHing it and the pipeline's execution.
    ...             # our best bet is to just retry.
    ...             continue

Note that, because the Pipeline must bind to a single connection for the
duration of a WATCH, care must be taken to ensure that the connection is
returned to the connection pool by calling the reset() method. If the
Pipeline is used as a context manager (as in the example above) reset()
will be called automatically. Of course you can do this the manual way by
explicitly calling reset():

.. code-block:: pycon

    >>> pipe = r.pipeline()
    >>> while True:
    ...     try:
    ...         pipe.watch('OUR-SEQUENCE-KEY')
    ...         ...
    ...         pipe.execute()
    ...         break
    ...     except WatchError:
    ...         continue
    ...     finally:
    ...         pipe.reset()

A convenience method named "transaction" exists for handling all the
boilerplate of handling and retrying watch errors. It takes a callable that
should expect a single parameter, a pipeline object, and any number of keys to
be WATCHed. Our client-side INCR command above can be written like this,
which is much easier to read:

.. code-block:: pycon

    >>> def client_side_incr(pipe):
    ...     current_value = pipe.get('OUR-SEQUENCE-KEY')
    ...     next_value = int(current_value) + 1
    ...     pipe.multi()
    ...     pipe.set('OUR-SEQUENCE-KEY', next_value)
    >>>
    >>> r.transaction(client_side_incr, 'OUR-SEQUENCE-KEY')
    [True]

Publish / Subscribe
^^^^^^^^^^^^^^^^^^^

redis-py includes a `PubSub` object that subscribes to channels and listens
for new messages. Creating a `PubSub` object is easy.

.. code-block:: pycon

    >>> r = redis.Redis(...)
    >>> p = r.pubsub()

Once a `PubSub` instance is created, channels and patterns can be subscribed
to.

.. code-block:: pycon

    >>> p.subscribe('my-first-channel', 'my-second-channel', ...)
    >>> p.psubscribe('my-*', ...)

The `PubSub` instance is now subscribed to those channels/patterns. The
subscription confirmations can be seen by reading messages from the `PubSub`
instance.

.. code-block:: pycon

    >>> p.get_message()
    {'pattern': None, 'type': 'subscribe', 'channel': 'my-second-channel', 'data': 1L}
    >>> p.get_message()
    {'pattern': None, 'type': 'subscribe', 'channel': 'my-first-channel', 'data': 2L}
    >>> p.get_message()
    {'pattern': None, 'type': 'psubscribe', 'channel': 'my-*', 'data': 3L}

Every message read from a `PubSub` instance will be a dictionary with the
following keys.

* **type**: One of the following: 'subscribe', 'unsubscribe', 'psubscribe',
  'punsubscribe', 'message', 'pmessage'
* **channel**: The channel [un]subscribed to or the channel a message was
  published to
* **pattern**: The pattern that matched a published message's channel. Will be
  `None` in all cases except for 'pmessage' types.
* **data**: The message data. With [un]subscribe messages, this value will be
  the number of channels and patterns the connection is currently subscribed
  to. With [p]message messages, this value will be the actual published
  message.

Let's send a message now.

.. code-block:: pycon

    # the publish method returns the number matching channel and pattern
    # subscriptions. 'my-first-channel' matches both the 'my-first-channel'
    # subscription and the 'my-*' pattern subscription, so this message will
    # be delivered to 2 channels/patterns
    >>> r.publish('my-first-channel', 'some data')
    2
    >>> p.get_message()
    {'channel': 'my-first-channel', 'data': 'some data', 'pattern': None, 'type': 'message'}
    >>> p.get_message()
    {'channel': 'my-first-channel', 'data': 'some data', 'pattern': 'my-*', 'type': 'pmessage'}

Unsubscribing works just like subscribing. If no arguments are passed to
[p]unsubscribe, all channels or patterns will be unsubscribed from.

.. code-block:: pycon

    >>> p.unsubscribe()
    >>> p.punsubscribe('my-*')
    >>> p.get_message()
    {'channel': 'my-second-channel', 'data': 2L, 'pattern': None, 'type': 'unsubscribe'}
    >>> p.get_message()
    {'channel': 'my-first-channel', 'data': 1L, 'pattern': None, 'type': 'unsubscribe'}
    >>> p.get_message()
    {'channel': 'my-*', 'data': 0L, 'pattern': None, 'type': 'punsubscribe'}

redis-py also allows you to register callback functions to handle published
messages. Message handlers take a single argument, the message, which is a
dictionary just like the examples above. To subscribe to a channel or pattern
with a message handler, pass the channel or pattern name as a keyword argument
with its value being the callback function.

When a message is read on a channel or pattern with a message handler, the
message dictionary is created and passed to the message handler. In this case,
a `None` value is returned from get_message() since the message was already
handled.

.. code-block:: pycon

    >>> def my_handler(message):
    ...     print 'MY HANDLER: ', message['data']
    >>> p.subscribe(**{'my-channel': my_handler})
    # read the subscribe confirmation message
    >>> p.get_message()
    {'pattern': None, 'type': 'subscribe', 'channel': 'my-channel', 'data': 1L}
    >>> r.publish('my-channel', 'awesome data')
    1
    # for the message handler to work, we need tell the instance to read data.
    # this can be done in several ways (read more below). we'll just use
    # the familiar get_message() function for now
    >>> message = p.get_message()
    MY HANDLER:  awesome data
    # note here that the my_handler callback printed the string above.
    # `message` is None because the message was handled by our handler.
    >>> print message
    None

If your application is not interested in the (sometimes noisy)
subscribe/unsubscribe confirmation messages, you can ignore them by passing
`ignore_subscribe_messages=True` to `r.pubsub()`. This will cause all
subscribe/unsubscribe messages to be read, but they won't bubble up to your
application.

.. code-block:: pycon

    >>> p = r.pubsub(ignore_subscribe_messages=True)
    >>> p.subscribe('my-channel')
    >>> p.get_message()  # hides the subscribe message and returns None
    >>> r.publish('my-channel', 'my data')
    1
    >>> p.get_message()
    {'channel': 'my-channel', 'data': 'my data', 'pattern': None, 'type': 'message'}

There are three different strategies for reading messages.

The examples above have been using `pubsub.get_message()`. Behind the scenes,
`get_message()` uses the system's 'select' module to quickly poll the
connection's socket. If there's data available to be read, `get_message()` will
read it, format the message and return it or pass it to a message handler. If
there's no data to be read, `get_message()` will immediately return None. This
makes it trivial to integrate into an existing event loop inside your
application.

.. code-block:: pycon

    >>> while True:
    >>>     message = p.get_message()
    >>>     if message:
    >>>         # do something with the message
    >>>     time.sleep(0.001)  # be nice to the system :)

Older versions of redis-py only read messages with `pubsub.listen()`. listen()
is a generator that blocks until a message is available. If your application
doesn't need to do anything else but receive and act on messages received from
redis, listen() is an easy way to get up an running.

.. code-block:: pycon

    >>> for message in p.listen():
    ...     # do something with the message

The third option runs an event loop in a separate thread.
`pubsub.run_in_thread()` creates a new thread and starts the event loop. The
thread object is returned to the caller of `run_in_thread()`. The caller can
use the `thread.stop()` method to shut down the event loop and thread. Behind
the scenes, this is simply a wrapper around `get_message()` that runs in a
separate thread, essentially creating a tiny non-blocking event loop for you.
`run_in_thread()` takes an optional `sleep_time` argument. If specified, the
event loop will call `time.sleep()` with the value in each iteration of the
loop.

Note: Since we're running in a separate thread, there's no way to handle
messages that aren't automatically handled with registered message handlers.
Therefore, redis-py prevents you from calling `run_in_thread()` if you're
subscribed to patterns or channels that don't have message handlers attached.

.. code-block:: pycon

    >>> p.subscribe(**{'my-channel': my_handler})
    >>> thread = p.run_in_thread(sleep_time=0.001)
    # the event loop is now running in the background processing messages
    # when it's time to shut it down...
    >>> thread.stop()

A PubSub object adheres to the same encoding semantics as the client instance
it was created from. Any channel or pattern that's unicode will be encoded
using the `charset` specified on the client before being sent to Redis. If the
client's `decode_responses` flag is set the False (the default), the
'channel', 'pattern' and 'data' values in message dictionaries will be byte
strings (str on Python 2, bytes on Python 3). If the client's
`decode_responses` is True, then the 'channel', 'pattern' and 'data' values
will be automatically decoded to unicode strings using the client's `charset`.

PubSub objects remember what channels and patterns they are subscribed to. In
the event of a disconnection such as a network error or timeout, the
PubSub object will re-subscribe to all prior channels and patterns when
reconnecting. Messages that were published while the client was disconnected
cannot be delivered. When you're finished with a PubSub object, call its
`.close()` method to shutdown the connection.

.. code-block:: pycon

    >>> p = r.pubsub()
    >>> ...
    >>> p.close()


The PUBSUB set of subcommands CHANNELS, NUMSUB and NUMPAT are also
supported:

.. code-block:: pycon

    >>> r.pubsub_channels()
    ['foo', 'bar']
    >>> r.pubsub_numsub('foo', 'bar')
    [('foo', 9001), ('bar', 42)]
    >>> r.pubsub_numsub('baz')
    [('baz', 0)]
    >>> r.pubsub_numpat()
    1204

Monitor
^^^^^^^
redis-py includes a `Monitor` object that streams every command processed
by the Redis server. Use `listen()` on the `Monitor` object to block
until a command is received.

.. code-block:: pycon

    >>> r = redis.Redis(...)
    >>> with r.monitor() as m:
    >>>     for command in m.listen():
    >>>         print(command)

Lua Scripting
^^^^^^^^^^^^^

redis-py supports the EVAL, EVALSHA, and SCRIPT commands. However, there are
a number of edge cases that make these commands tedious to use in real world
scenarios. Therefore, redis-py exposes a Script object that makes scripting
much easier to use.

To create a Script instance, use the `register_script` function on a client
instance passing the Lua code as the first argument. `register_script` returns
a Script instance that you can use throughout your code.

The following trivial Lua script accepts two parameters: the name of a key and
a multiplier value. The script fetches the value stored in the key, multiplies
it with the multiplier value and returns the result.

.. code-block:: pycon

    >>> r = redis.Redis()
    >>> lua = """
    ... local value = redis.call('GET', KEYS[1])
    ... value = tonumber(value)
    ... return value * ARGV[1]"""
    >>> multiply = r.register_script(lua)

`multiply` is now a Script instance that is invoked by calling it like a
function. Script instances accept the following optional arguments:

* **keys**: A list of key names that the script will access. This becomes the
  KEYS list in Lua.
* **args**: A list of argument values. This becomes the ARGV list in Lua.
* **client**: A redis-py Client or Pipeline instance that will invoke the
  script. If client isn't specified, the client that initially
  created the Script instance (the one that `register_script` was
  invoked from) will be used.

Continuing the example from above:

.. code-block:: pycon

    >>> r.set('foo', 2)
    >>> multiply(keys=['foo'], args=[5])
    10

The value of key 'foo' is set to 2. When multiply is invoked, the 'foo' key is
passed to the script along with the multiplier value of 5. Lua executes the
script and returns the result, 10.

Script instances can be executed using a different client instance, even one
that points to a completely different Redis server.

.. code-block:: pycon

    >>> r2 = redis.Redis('redis2.example.com')
    >>> r2.set('foo', 3)
    >>> multiply(keys=['foo'], args=[5], client=r2)
    15

The Script object ensures that the Lua script is loaded into Redis's script
cache. In the event of a NOSCRIPT error, it will load the script and retry
executing it.

Script objects can also be used in pipelines. The pipeline instance should be
passed as the client argument when calling the script. Care is taken to ensure
that the script is registered in Redis's script cache just prior to pipeline
execution.

.. code-block:: pycon

    >>> pipe = r.pipeline()
    >>> pipe.set('foo', 5)
    >>> multiply(keys=['foo'], args=[5], client=pipe)
    >>> pipe.execute()
    [True, 25]

Sentinel support
^^^^^^^^^^^^^^^^

redis-py can be used together with `Redis Sentinel <https://redis.io/topics/sentinel>`_
to discover Redis nodes. You need to have at least one Sentinel daemon running
in order to use redis-py's Sentinel support.

Connecting redis-py to the Sentinel instance(s) is easy. You can use a
Sentinel connection to discover the master and slaves network addresses:

.. code-block:: pycon

    >>> from redis.sentinel import Sentinel
    >>> sentinel = Sentinel([('localhost', 26379)], socket_timeout=0.1)
    >>> sentinel.discover_master('mymaster')
    ('127.0.0.1', 6379)
    >>> sentinel.discover_slaves('mymaster')
    [('127.0.0.1', 6380)]

You can also create Redis client connections from a Sentinel instance. You can
connect to either the master (for write operations) or a slave (for read-only
operations).

.. code-block:: pycon

    >>> master = sentinel.master_for('mymaster', socket_timeout=0.1)
    >>> slave = sentinel.slave_for('mymaster', socket_timeout=0.1)
    >>> master.set('foo', 'bar')
    >>> slave.get('foo')
    'bar'

The master and slave objects are normal Redis instances with their
connection pool bound to the Sentinel instance. When a Sentinel backed client
attempts to establish a connection, it first queries the Sentinel servers to
determine an appropriate host to connect to. If no server is found,
a MasterNotFoundError or SlaveNotFoundError is raised. Both exceptions are
subclasses of ConnectionError.

When trying to connect to a slave client, the Sentinel connection pool will
iterate over the list of slaves until it finds one that can be connected to.
If no slaves can be connected to, a connection will be established with the
master.

See `Guidelines for Redis clients with support for Redis Sentinel
<https://redis.io/topics/sentinel-clients>`_ to learn more about Redis Sentinel.

Scan Iterators
^^^^^^^^^^^^^^

The \*SCAN commands introduced in Redis 2.8 can be cumbersome to use. While
these commands are fully supported, redis-py also exposes the following methods
that return Python iterators for convenience: `scan_iter`, `hscan_iter`,
`sscan_iter` and `zscan_iter`.

.. code-block:: pycon

    >>> for key, value in (('A', '1'), ('B', '2'), ('C', '3')):
    ...     r.set(key, value)
    >>> for key in r.scan_iter():
    ...     print key, r.get(key)
    A 1
    B 2
    C 3

Author
^^^^^^

redis-py is developed and maintained by Andy McCurdy (sedrik@gmail.com).
It can be found here: https://github.com/andymccurdy/redis-py

Special thanks to:

* Ludovico Magnocavallo, author of the original Python Redis client, from
  which some of the socket code is still used.
* Alexander Solovyov for ideas on the generic response callback system.
* Paul Hubbard for initial packaging support.

