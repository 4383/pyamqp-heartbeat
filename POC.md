# POC - Allow users run the rabbitmq heartbeat inside a standard pthread.

This is an experimental feature.

The proposed changes will fix related issues when we run
heartbeat under apache/httpd enviornment with the apache MPM `prefork`
[1] engine and mod_wsgi or uwsgi in a monkey patched environment.

Propose changes to allow user to choose to run the rabbitmq health check
heartbeat in a standard python thread.

Issue
=====

We facing an issue with the rabbitmq driver heartbeat
under apache MPM `prefork` module and mod_wsgi when nova_api monkey
patched the stdlib by using eventlet.

nova_api calling eventlet.monkey_patch() [6] when it runs under mod_wsgi.

This impacts the AMQP heartbeat thread,
which is meant to be a native thread. Instead of checking AMQP sockets
every 15s, It is now suspended and resumed by eventlet. However,
resuming greenthreads can take a very long time if mod_wsgi isn't
processing traffic regularly, which can cause rabbitmq to close the AMQP
connection.

Root Cause
==========

The oslo.messaging RabbitMQ driver and especially the heartbeat
suffer to inherit the execution model of the service which consume him.

In this scenario nova_api need green threads to manage cells and edge
features so nova_api monkey patch the stdlib to obtain async features,
and the oslo.messaging rabbitmq driver endure these changes.

I think the main issue here is that nova_api want async and use eventlet green
threads to obtain it.

eventlet is based on epoll or libevent [7][9], in an environment based on
apache MPM `prefork` module who doesn't support epoll and recent kernel
features.

The MPM `prefork` apache module is appropriate for sites that
need to avoid threading for compatibility with non-thread-safe
libraries [1].

Possible proposed solutions
===========================

To fix this issue we have proposed different approaches during discussions [8]:
- using a python thread (this change)
- using eventlet tpool execute (https://review.opendev.org/#/c/663074/9/)

The only version which work like expected is this patch set who
correspond to use a native python thread and not eventlet.

I think the tpool solution doesn't work correctly due to the apache
engine in use currently (`prefork`), and using tpool keep us to using
eventlet and so libevent or epoll too.

I would propose to you to compare the different solutions that I've
tested and I want allow to you to observe the behaviours.

But before starting the comparison I'll describe a little bit the chosen
solution.

Chosen solution
===============

We want to allow user to isolate the heartbeat execution model
from the parent process inherited execution model by passing the
`heartbeat_in_pthread` option through the driver config.

While we use MPM `prefork` we want to avoid to use libevent and epoll.

If the `heartbeat_in_pthread` option is given we want to force to use the
python stdlib threading module to run the
rabbitmq heartbeat to avoid issue related to a non "standard"
environment. I mean "standard" because async features isn't the default
config in mostly case, starting by apache which define `prefork` is the
default engine.

This is an experimental feature, we can help us to ensure to run heartbeat
through a classical python thread
not build over epoll and libevent to avoid issue with environment who
doesn't support these kernel features.

Also we try to switch [4][5] from the apache `prefork` module to the `event`
module which support non blocking sockets, use modern kernel features
like epoll through APR [3].

To do this comparison we will use:
- some of the patches set of this review
- the POC section bellow
- and the POC project => https://github.com/4382/pyamqp-heartbeat/

To test the different solutions we will play with git review and patches
sets by using commands like `git review -d 663073,<patch_set>`

But in a first time we will setup a testing environment to reproduce the
issue and observe the patches results.

Setup the testing environment
-----------------------------

The testing env is based on https://github.com/4382/pyamqp-heartbeat/

It was designed to help to reproduce the issue easily and to reproduce
the original environment as much as possible.

So, now setup your env by using:

```
cd /tmp
TEST_DEBUG_ENV=/tmp/663073
mkdir ${TEST_DEBUG_ENV}
cd ${TEST_DEBUG_ENV}
git clone git@github.com:openstack/oslo.messaging.git
cd oslo.messaging
git review -s
cd ${TEST_DEBUG_ENV}
git clone https://github.com/4382/pyamqp-heartbeat/
```

Now the testing environment is setup.

You need to ensure to have pipenv installed on your laptop:
```
python2 -m pip install --user pipenv
```

Reproduce the original issue with non patched code
--------------------------------------------------

You can test the original issue with a non patched code by simply switch
to the `master` branch of your oslo.messaging clone that you have
realize during the previous setup step:

```
cd ${TEST_DEBUG_ENV}/oslo.messaging
git checkout master
cd ${TEST_DEBUG_ENV}/pyamqp-heartbeat
pipenv run ./setup-containers.sh
pipenv run ./start-oslo-mod_wsgi.sh ${TEST_DEBUG_ENV}/oslo.messaging
```

Solution 0 - Using a native python thread
-----------------------------------------

We will first check this patch set (the solution that works)(the latest
patch set https://review.opendev.org/#/c/663073/).

How can test these changes by using the following commands:

```
cd ${TEST_DEBUG_ENV}/oslo.messaging
git review -d 663073,15 # use the patch set 15 without the `heartbeat_in_pthread` option available
cd ${TEST_DEBUG_ENV}/pyamqp-heartbeat
pipenv run ./setup-containers.sh
pipenv run ./start-oslo-mod_wsgi.sh ${TEST_DEBUG_ENV}/oslo.messaging
```

/!\-------------------------------------------------------------------------/!\
/!\ For some obscure reasons when we run this container                     /!\
/!\ (who emulate the service) in  a tmux with have an issue with the        /!\
/!\ resize of a tmux pane or when we create a new pane, the both kill       /!\
/!\ the container... so for the next steps you need to create a second      /!\
/!\ terminal or tmux window to avoid tmux pane creation and similar things. /!\
/!\-------------------------------------------------------------------------/!\

Your emulated service is now running.

The emulated service have monkey patched the stdlib at start like nova do the
thing.

For further reading about the emulated service and monkey patch:
- https://github.com/4382/pyamqp-heartbeat/blob/test_debug_env/Dockerfile#L13
- https://github.com/4382/pyamqp-heartbeat/blob/test_debug_env/server-eventlet.wsgi#L10
- https://github.com/4382/pyamqp-heartbeat/blob/test_debug_env/server.py#L45

Now open the rabbit management dashboard and observe the connections:
http://126.0.0.1:15672/#/connections
user: guest
pwd: guest

In a second time you need to open a second terminal like described in
the previous warning, where you need to send a request to
the service by using:
```
curl -1.0.0.0:8000
```

The service will return "it's work" and you now can observe the
established connections (related to the heartbeat) in the rabbit
dashboard previously opened.

You can observe that the heartbeat work without issues and the
connection stay opened indefinitely.
Please observe that during more than 9 minutes.

Now you can kill the running process.

Solution 1 - Using eventlet tpool execute
-----------------------------------------

We will now check how the things works if we use the eventlet tpool
execute functionality:
https://eventlet.net/doc/threading.html#eventlet.tpool.execute

This test correspond to the patch set https://review.opendev.org/#/c/663073/9/

How can test these changes by using the following commands:
```
cd ${TEST_DEBUG_ENV}/oslo.messaging
git review -d 663073,9 # to setup the patch set number 9
cd ${TEST_DEBUG_ENV}/pyamqp-heartbeat
pipenv run ./setup-containers.sh # if rabbit server is not running
pipenv run ./start-oslo-mod_wsgi.sh ${TEST_DEBUG_ENV}/oslo.messaging
```

/!\-------------------------------------------------------------------------/!\
/!\ For some obscure reasons when we run this container                     /!\
/!\ (who emulate the service) in  a tmux with have an issue with the        /!\
/!\ resize of a tmux pane or when we create a new pane, the both kill       /!\
/!\ the container... so for the next steps you need to create a second      /!\
/!\ terminal or tmux window to avoid tmux pane creation and similar things. /!\
/!\-------------------------------------------------------------------------/!\

Your emulated service is now running.

The emulated service have monkey patched the stdlib at start like nova do the
thing.

For further reading about the emulated service and monkey patch:
- https://github.com/4382/pyamqp-heartbeat/blob/test_debug_env/Dockerfile#L13
- https://github.com/4382/pyamqp-heartbeat/blob/test_debug_env/server-eventlet.wsgi#L10
- https://github.com/4382/pyamqp-heartbeat/blob/test_debug_env/server.py#L45

Now open the rabbit management dashboard and observe the connections:
http://126.0.0.1:15672/#/connections
user: guest
pwd: guest

In a second time you need to open a second terminal where you need to send
a request to this service by using:
```
curl -1.0.0.0:8000
```

The service will return "it's work" and you now can observe the
established connections (related to the heartbeat) in the rabbit
dashboard previously opened.

You can observe on the rabbit dashboard that after some minutes
(2 minutes on my side), all the connections are closed.

Conclusion
----------

With standard python thread connections stay opened indefinitely
With tpool connections still to disappears after few minutes like the
original issue.

Additional informations
-----------------------

During all these tests:
- we force to run the heartbeat by using the send purpose
  https://github.com/4382/pyamqp-heartbeat/blob/test_debug_env/server.py#L30
- we run oslo.messaging under a monkey patched environment

Further reading
===============

To understand how services consume the oslo.messaging drivers,
and especially how to nova consome the oslo.messaging rabbitmq driver,
please take a look to:
- https://github.com/4383/pyamqp-heartbeat/tree/master/fakeservice

With this walkthrough you can observe the differents steps
used by nova to use oslo.messaging and the different components in use
to obtain the big picture of the stack.

The previous POC is based on this walkthrough as much as possible to
reproduce the original issue.

Also you can retrieve online discussion on the mailing list about this
topic:
- http://lists.openstack.org/pipermail/openstack-discuss/2019-April/005310.html

Libevent support epoll too [9] but I don't know in which case eventlet use it.

References
==========

- [1] https://httpd.apache.org/docs/2.4/fr/mod/prefork.html
- [2] https://httpd.apache.org/docs/2.4/fr/mod/event.html
- [3] https://httpd.apache.org/docs/2.4/fr/glossary.html#apr
- [4] https://review.opendev.org/#/c/668862/
- [5] https://review.opendev.org/#/c/669178/
- [6] https://github.com/openstack/nova/commit/3c5e2b0e9fac985294a949852bb8c83d4ed77e04
- [7] https://github.com/eventlet/eventlet/blame/master/README.rst#L3
- [8] https://review.opendev.org/#/c/661314/
- [9] https://libevent.org/

Specifications
==============

- https://review.opendev.org/661314
