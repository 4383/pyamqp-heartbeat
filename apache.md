https://bugs.launchpad.net/nova/+bug/1829062/comments/7
https://uwsgi-docs.readthedocs.io/en/latest/Options.html#threads
https://httpd.apache.org/docs/2.4/mod/prefork.html
https://httpd.apache.org/docs/2.4/mod/worker.html
https://bugs.launchpad.net/nova/+bug/856764
https://review.opendev.org/#/c/657168/ohttps://review.opendev.org/#/c/662095/3/releasenotes/notes/eventlet-monkey-patch-5f734ef581aa550e.yaml

- [puppet-wsgi-apache-threads = 1](https://github.com/openstack/puppet-openstacklib/blob/master/manifests/wsgi/apache.pp#L86)
- [puppet nova threads](https://github.com/openstack/puppet-nova/blob/master/manifests/wsgi/apache_api.pp#L152)
- [apache mpm worker](https://httpd.apache.org/docs/2.4/mod/worker.html)
- [apache mpm worker ThreadsPerChild](https://httpd.apache.org/docs/2.4/mod/mpm_common.html#threadsperchild)
- [/!\ apache mpm prefork /!\\](https://httpd.apache.org/docs/2.4/mod/prefork.html) is the [default mode](https://modwsgi.readthedocs.io/en/develop/user-guides/processes-and-threading.html?highlight=idle#the-unix-prefork-mpm)
- [apache mpm common - maxsparethreads](https://httpd.apache.org/docs/2.4/mod/mpm_common.html#maxsparethreads)
- [apache event](https://httpd.apache.org/docs/2.4/mod/event.html)
- [mod_wsgi IDLE - restart-interval](https://modwsgi.readthedocs.io/en/develop/release-notes/version-4.5.12.html?highlight=idle)
- [mod_wsgi IDLE - eviction-timeout](https://modwsgi.readthedocs.io/en/develop/release-notes/version-4.4.7.html?highlight=idle)
- [mod_wsgi IDLE - WSGIIgnoreActivity](https://modwsgi.readthedocs.io/en/develop/release-notes/version-4.5.8.html?highlight=idle)
- [mod_wsgi IDLE - WSGIDaemonProcess](https://modwsgi.readthedocs.io/en/develop/configuration-directives/WSGIDaemonProcess.html?highlight=idle)
- [mod_wsgi IDLE - processes and threading](https://modwsgi.readthedocs.io/en/develop/user-guides/processes-and-threading.html?highlight=idle)
- [thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)
- [horizon apache switch from prefork to event](https://review.opendev.org/#/c/72666/)
- [MPM engine summarize](https://www.valinv.com/dev/apache-apache-prefork-worker-event)
- [MPM event perfs stackoverflow](https://stackoverflow.com/questions/27856231/why-is-the-apache-event-mpm-performing-poorly)
- [apache 2.4 vulnerability threads](https://www.cvedetails.com/cve/CVE-2019-0211/)

```
2019-05-27 10:00:18	bogdando	hi
2019-05-27 10:00:25	bogdando	dciabrin: seems like we need some tuning for Apache 2 MPM https://bugs.launchpad.net/nova/+bug/1829062/comments/7
2019-05-27 10:09:03	dciabrin	hey bogdando, yep we had a chat last friday with the nova folks about mixing thread and eventlet
2019-05-27 10:10:37	dciabrin	bogdando, mod_wsgi needs to use 1 thread when eventlet is in use, which I believe is already the case for the apache config in nova_api
2019-05-27 10:12:57	dciabrin	bogdando, we also agreed 1) short-term to disable the heartbeat thread in nova_api and rely on tcp keepalives 2) update oslo_messaging to allow to create a real thread (as opposed to a greenthread when eventlet is in use) for the amqp heartbeat thread, to avoid slow thread scheduling issue and amqp disconnection
2019-05-27 10:16:56	bogdando	dciabrin: I thought that comment was about forcing wsgi to use processes instead of threads? So there shouldn't be situations when a single interpreter started by wsgi runs a 2 threads? So I got that message as   tuning the Apache 2 mpm configs https://httpd.apache.org/docs/2.4/mpm.html as appropriate
2019-05-27 10:17:21	dciabrin	bogdando, right that comment is about that indeed. i'm unclear
2019-05-27 10:18:16	bogdando	dciabrin: > disable the heartbeat thread in nova_api and rely on tcp keepalives
2019-05-27 10:18:16	bogdando	that really makes RPC corner cases handled poorly
2019-05-27 10:18:24	bogdando	so failover just sucks )
2019-05-27 10:18:25	dciabrin	bogdando, we had a bjn with the nova folks to discuss the problem with the amqp heartbeat thread (not sure if you're aware?). and during this meeting smooney made it clear that nova_api needed to configure concurrency via processes and always use 1 thread, otherwise eventlet will complain
2019-05-27 10:19:09	dciabrin	bogdando, hmm can you elaborate? do you see any particular condition when that worsen failover?
2019-05-27 10:19:38	bogdando	dciabrin: nope, not aware of the details of that conversations. Just have some memories for the reasons of introducing heartbeats in opernstack exactly for that, to not mess with TCP ka
2019-05-27 10:20:00	bogdando	dciabrin: only long bearded bugs
2019-05-27 10:20:40	dciabrin	bogdando, so as i said, that is a short term move to workaround the fact that currecntly amqp heartbeat threads are broken in nova_api in stein
2019-05-27 10:21:25	dciabrin	bogdando, i tried to summarize the issue in https://review.opendev.org/#/c/657168/, last comment
2019-05-27 10:21:43	bogdando	right, just that it looks like those  2 conclusions did not account the recent update for mpm
2019-05-27 10:22:17	bogdando	I'd try it with https://httpd.apache.org/docs/2.4/mod/prefork.html instead perhaps
2019-05-27 10:24:11	dciabrin	ah ok. well i _think_ we already had the good config in tripleo for nova_api (multiple processes, 1 thread). mschuppert confirmed that in https://bugzilla.redhat.com/show_bug.cgi?id=1706456#c3
2019-05-27 10:24:12	bhjf	Bug 1706456: high, unspecified, ---, ---, mschuppe, openstack-tripleo-heat-templates, NEW , [OSP15] AMQP heartbeat thread missing heartbeats when running under nova_api
2019-05-27 10:25:55	bogdando	dciabrin: anyway, I'd recommend to verify outcomes of disabling heartbeat thread in nova_api for tcp keepalives very carefully, with an advanced destructive/failover testing
2019-05-27 10:26:06	dciabrin	+1
2019-05-27 10:26:12	bogdando	IIRC, that's https://bugs.launchpad.net/nova/+bug/856764 was the original issue
2019-05-27 10:26:19	bogdando	will look for more
2019-05-27 10:27:11	dciabrin	bogdando, oh thx for the link!
2019-05-27 10:27:44	bogdando	yea, so the pattern of a failure would be seeing greatly increased "Timed out waiting for a reply" everywhere :)
2019-05-27 10:30:20	dciabrin	hberaud, ^^ fyi, we probably want to come up with a quick solution to force oslo messagin to create a real pthread for the amqp heartbeats
2019-05-27 10:30:57	hberaud	dciabrin: ack
2019-05-27 10:32:33	hberaud	dciabrin: I guess the most quickly solution on my side is the described on https://review.opendev.org/#/c/661314/ (deactivate eventlet and switch on pthread with config argument)
2019-05-27 10:33:16	hberaud	dciabrin: another can be in a second time to introduce futurist threadpool or something like that 
2019-05-27 10:33:27	hberaud	to avoid eventlet
2019-05-27 10:34:33	dciabrin	hberaud, ack ok I'll add some comments into that review
2019-05-27 10:34:38	dciabrin	thx!
2019-05-27 10:35:06	hberaud	dciabrin: thanks don't hesitate
2019-05-27 10:48:25	dciabrin	hberaud, ah you beat me at answering to the review. I left my 2c though
```

```
11:10:30  hberaud | how are you?
11:11:02 bogdando | I'm fine thanks, and how are you?
11:11:42  hberaud | I'm fine thanks. Can I disturb you few minutes about apache mpm and threading things etc... (related to the heartbeat issue).
11:12:06 bogdando | sure thing
11:13:42 bogdando | that's for https://bugs.launchpad.net/nova/+bug/1829062/comments/7 , right?
11:14:07  hberaud | I've writing a walkthrough on nova consume oslo.messaging and heartbeat things, from the puppet config of nova to the instanciation of the oslo.messaging rabbitmq driver
                  | (https://github.com/dciabrin/pyamqp-heartbeat/tree/master/fakeservice)
11:14:13  hberaud | (sorry)
11:14:16  hberaud | yes
11:14:55  hberaud | so first if you have spare time can you take a look to this walkthrough and correct me if I'm wrong on some parts
11:15:15  hberaud | second, now I inspect how apache work deeply etc...
11:15:30 bogdando | will do
11:15:46  hberaud | and you've tell to us few months ago that you want to test the prefork module
11:16:01 bogdando | although I'm not great in deep understanding of the details
11:16:28 bogdando | for those rpc and wsgi things
11:16:29 bogdando | while the writing looks quite thorough
11:16:30  hberaud | have you test the prefork module?
11:17:20 bogdando | prefork, right. IIUC we had experience with it in fuel
11:17:22  hberaud | I've start a second walkthrough more focused on apache and apache module to understand why eventlet is an issue
11:17:26 bogdando | in the past
11:18:09  hberaud | and the prefork seem different with threads than the worker module https://httpd.apache.org/docs/2.4/mod/worker.html
11:18:24  hberaud | I'm not an httpd expert
11:19:09  hberaud | I just try to understand the execution life cycle of each pieces of the puzzle to try to propose a support of eventlet or something like that
11:22:20 _hberaud | I can see that this kind of issue occur since few years ago now on openstack and in different services/manner etc... so I think the only real solution is to fix the thing on the eventlet or
                  | httpd side, to avoid mole-whacking
11:22:50 bogdando | did you get my recent messages?
11:22:54 _hberaud | (sorry I was deconnected from irc)
11:22:58 _hberaud | no
11:23:08 bogdando | for the passed ~6 min
11:23:25 _hberaud | last msg 11:17:26 bogdando | in the past
11:23:48 bogdando | 11:18:24 - bogdando: I could help with some code and testing, but would prefer doing that as a pair programming exercise - if you could prepare the env with infra-red etc...
11:23:48 bogdando | 11:18:38 - bogdando: (I know nothing about infra-red :) )
11:23:48 bogdando | 11:18:46 - bogdando: hope that works?
11:23:48 bogdando | 11:21:14 - bogdando: here is a great example  https://review.opendev.org/#/c/72666/
11:23:48 bogdando | 11:21:25 - bogdando: for tuning wsgi processes/threads for mpm
11:23:48 bogdando | 11:21:37 - bogdando: but swutching from prefork to event
11:23:48 bogdando | 11:21:53 - bogdando: (so my memories have failed me, sorry)
11:24:23 _hberaud | nice
11:24:57 bogdando | 11:24:14 - bogdando: so we prolly need that patch adapted for 1 to many or many to 1 mapping?
11:24:57 bogdando | 11:24:32 - bogdando: but surely not N:N
11:25:08 bogdando | yea processes=<api workers> threads=1
11:25:26 bogdando | should not be hard to adapt those for puppet-tripleo
11:25:53 _hberaud | make sense, I will try to collect the maximum of info and in a second time I'll initiate a testing env and ping you
11:26:06 bogdando | k
11:26:14 _hberaud | sound good for you?
11:26:48 bogdando | yea
11:27:25 _hberaud | I also plan to post my walkthroughs to the openstack ML to share my knowledge about all these things, I think it's can be useful for many people
11:27:48 _hberaud | so thanks for your time :)
11:28:14 bogdando | so we could try to apply something very similar to nova api ad hoc, then compose a patch for puppets
11:28:58 _hberaud | yeah sure
11:30:02 bogdando | I'll draft something right now for the upstream placeholder patch
11:30:10 _hberaud | I will study https://review.opendev.org/#/c/72666/
11:30:34 _hberaud | ack do not hesitate to ping me if you want something (help, review, code, etc...)
```
