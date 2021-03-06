.. _standalone:

Servers
=======

An example of the simplest server was presented in the :ref:`introduction`. A single server running
in a single process may be useful, but there are a million alternatives to pylm for that. The real usefulness
of pylm arrives when the workload is so large that a single server is not capable of handling it. Here we introduce
parallelism for the first time with the parallel standalone server.

A simple picture of the architecture of a parallel server is presented in the next figure. The client
connects to a master process that manages an arbitrary number of workers. The workers act as slaves, and connect
only to the master.


.. only:: html

    .. figure:: _images/parallel.png
        :align: center

.. only:: latex

    .. figure:: _images/parallel.pdf
        :align: center
        :scale: 60

The following is a simple example on how to configure and run a parallel server. Since the parallel server
is designed to handle a large workload, the job method of the client expects a generator that creates a
series of binary messages.

.. literalinclude:: ../../examples/parallel/master.py
    :language: python
    :linenos:

.. literalinclude:: ../../examples/parallel/worker.py
    :language: python
    :linenos:

.. literalinclude:: ../../examples/parallel/client.py
    :language: python
    :linenos:

The master can be run as follows::

    $> python master.py

And we can launch two workers as follows::

    $> python worker.py worker1
    $> python worker.py worker2

Finally, here's how to run the client and its output::

    $> python client.py
    b'worker2 processed a message'
    b'worker1 processed a message'
    b'worker2 processed a message'
    b'worker1 processed a message'
    b'worker2 processed a message'
    b'worker1 processed a message'
    b'worker2 processed a message'
    b'worker1 processed a message'
    b'worker2 processed a message'
    b'worker1 processed a message'

The communication between the master and the workers is a PUSH-PULL queue of ZeroMQ. This means
that the most likely distribution pattern between the master and the workers follows a round-robin
scheduling.

Again, this simple example shows very little of the capabilities of this pattern in pylm. We'll introduce
features step by step creating a manager with more and more capabilities.

Cache
-----

One of the services that the master offers is a small key-value database that **can be seen by all the workers**.
You can use that database with RPC-style using :py:meth:`pylm.clients.Client.set`,
:py:meth:`pylm.clients.Client.get`, and :py:meth:`pylm.clients.Client.delete` methods.
Like the messages, the data to be stored in the database must be binary.

.. note::

   Note that the calling convention of :py:meth:`pylm.clients.Client.set` is not that conventional.
   Remember to pass first the value, and then the key if you want to use your own.

.. important::

   The master stores the data in memory. Have that in mind if you plan to send lots of data to the master.


The following example is a little modification from the previous example. The client, previously to sending
the job, it sets a value in the temporary cache of the master server. The workers, where the value of the
cached variable is hardcoded within the function that is executed, get the value and they use it to build the
response. The variations respect to the previous examples have been empasized.

.. literalinclude:: ../../examples/cache/master.py
    :language: python
    :linenos:

.. literalinclude:: ../../examples/cache/worker.py
    :language: python
    :linenos:
    :emphasize-lines: 7,8

.. literalinclude:: ../../examples/cache/client.py
    :language: python
    :linenos:
    :emphasize-lines: 10,11

And the output is the following::

    $> python client.py
    b' cached data '
    b'worker1 cached data a message'
    b'worker2 cached data a message'
    b'worker1 cached data a message'
    b'worker2 cached data a message'
    b'worker1 cached data a message'
    b'worker2 cached data a message'
    b'worker1 cached data a message'
    b'worker2 cached data a message'
    b'worker1 cached data a message'
    b'worker2 cached data a message'


Scatter messages from the master to the workers
-----------------------------------------------

Master server has a useful method called :py:meth:`pylm.servers.Master.scatter`, that
is in fact a generator. For each message that the master gets from the inbound socket, this
generator is executed. It is useful to modify the message stream in any conceivable way. In the
following example, right at the highlighted lines, a new master server overrides this ``scatter``
generator with a new one that sends the message it gets three times.

.. literalinclude:: ../../examples/scatter/master.py
    :language: python
    :linenos:
    :emphasize-lines: 4,5,6,7,8

The workers are identical than in the previous example. Since each message that the client sends
to the master is repeated three times, the client expects 30 messages instead of 10.

.. literalinclude:: ../../examples/scatter/client.py
    :language: python
    :linenos:
    :emphasize-lines: 30

This is the (partially omitted) output of the client::

    $> python client.py
    b' cached data '
    b'worker1 cached data a message'
    b'worker1 cached data a message'
    b'worker2 cached data a message'

    ...

    b'worker1 cached data a message'
    b'worker2 cached data a message'
    b'worker1 cached data a message'
    b'worker2 cached data a message'

Gather messages from the workers
--------------------------------

You can also alter the message stream after the workers have done their job. The master server also includes a
:py:meth:`pylm.servers.Master.gather` method, that is a generator too, that is executed for each message.
Being a generator, this means that gather has to yield, and that the result can be either no message, or an arbitrary
amount of messages. To make this example a little more interesting, we will also disassemble one of these messages that
were apparently just a bunch of bytes.

We will define a gather generator that counts the amount of messages, and when the message number 30 arrives,
the final message, its payload is changed with a different binary string. This means that we need to add an attribute
to the server for the counter, and we have to modify a message with :py:meth:`pylm.servers.Master.change_payload`.

See that this example is incremental respect to the previous one, and in consequence it uses the cache service and
the scatter and the gather generators.

.. literalinclude:: ../../examples/gather/master.py
    :language: python
    :linenos:
    :emphasize-lines: 6-8, 14-20

::

    $> python client.py
    b' cached data '
    b'worker1 cached data a message'
    b'worker2 cached data a message'
    b'worker2 cached data a message'

    ...

    b'worker2 cached data a message'
    b'worker1 cached data a message'
    b'worker2 cached data a message'
    b'worker1 cached data a message'
    b'Final message'

