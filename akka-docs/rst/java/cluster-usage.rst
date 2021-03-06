
.. _cluster_usage_java:

######################
 Cluster Usage
######################

.. note:: This module is :ref:`experimental <experimental>`. This document describes how to use the features implemented so far. More features are coming in Akka Coltrane. Track progress of the Coltrane milestone in `Assembla <http://www.assembla.com/spaces/akka/tickets>`_ and the `Roadmap <https://docs.google.com/document/d/18W9-fKs55wiFNjXL9q50PYOnR7-nnsImzJqHOPPbM4E/edit?hl=en_US>`_.

For introduction to the Akka Cluster concepts please see :ref:`cluster`.

Preparing Your Project for Clustering
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The Akka cluster is a separate jar file. Make sure that you have the following dependency in your project::

  <dependency>
    <groupId>com.typesafe.akka</groupId>
    <artifactId>akka-cluster-experimental_@binVersion@</artifactId>
    <version>@version@</version>
  </dependency>

If you are using the latest nightly build you should pick a timestamped Akka
version from
`<http://repo.typesafe.com/typesafe/snapshots/com/typesafe/akka/akka-cluster-experimental_@binVersion@/>`_.
We recommend against using ``SNAPSHOT`` in order to obtain stable builds.

.. _cluster_simple_example_java:

A Simple Cluster Example
^^^^^^^^^^^^^^^^^^^^^^^^

The following small program together with its configuration starts an ``ActorSystem``
with the Cluster enabled. It joins the cluster and logs some membership events.

Try it out:

1. Add the following ``application.conf`` in your project, place it in ``src/main/resources``:


.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/resources/application.conf#cluster

To enable cluster capabilities in your Akka project you should, at a minimum, add the :ref:`remoting-java`
settings, but with ``akka.cluster.ClusterActorRefProvider``.
The ``akka.cluster.seed-nodes`` should normally also be added to your ``application.conf`` file.

The seed nodes are configured contact points for initial, automatic, join of the cluster.

Note that if you are going to start the nodes on different machines you need to specify the
ip-addresses or host names of the machines in ``application.conf`` instead of ``127.0.0.1``

2. Add the following main program to your project, place it in ``src/main/java``:

.. literalinclude:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/simple/japi/SimpleClusterApp.java
   :language: java

3. Start the first seed node. Open a terminal window and run (one line)::

    mvn exec:java -Dexec.mainClass="sample.cluster.simple.japi.SimpleClusterApp" \
      -Dexec.args="2551"

2551 corresponds to the port of the first seed-nodes element in the configuration.
In the log output you see that the cluster node has been started and changed status to 'Up'.

4. Start the second seed node. Open another terminal window and run::

    mvn exec:java -Dexec.mainClass="sample.cluster.simple.japi.SimpleClusterApp" \
      -Dexec.args="2552"


2552 corresponds to the port of the second seed-nodes element in the configuration.
In the log output you see that the cluster node has been started and joins the other seed node
and becomes a member of the cluster. Its status changed to 'Up'.

Switch over to the first terminal window and see in the log output that the member joined.

5. Start another node. Open a maven session in yet another terminal window and run::

    mvn exec:java -Dexec.mainClass="sample.cluster.simple.japi.SimpleClusterApp"

Now you don't need to specify the port number, and it will use a random available port.
It joins one of the configured seed nodes. Look at the log output in the different terminal
windows.

Start even more nodes in the same way, if you like.

6. Shut down one of the nodes by pressing 'ctrl-c' in one of the terminal windows.
The other nodes will detect the failure after a while, which you can see in the log
output in the other terminals.

Look at the source code of the program again. What it does is to create an actor
and register it as subscriber of certain cluster events. It gets notified with
an snapshot event, ``CurrentClusterState`` that holds full state information of
the cluster. After that it receives events for changes that happen in the cluster.

Automatic vs. Manual Joining
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You may decide if joining to the cluster should be done automatically or manually.
By default it is automatic and you need to define the seed nodes in configuration
so that a new node has an initial contact point. When a new node is started it
sends a message to all seed nodes and then sends join command to the one that
answers first. If no one of the seed nodes replied (might not be started yet)
it retries this procedure until successful or shutdown.

The seed nodes can be started in any order and it is not necessary to have all
seed nodes running, but the node configured as the first element in the ``seed-nodes``
configuration list must be started when initially starting a cluster, otherwise the 
other seed-nodes will not become initialized and no other node can join the cluster. 
It is quickest to start all configured seed nodes at the same time (order doesn't matter), 
otherwise it can take up to the configured ``seed-node-timeout`` until the nodes
can join.

Once more than two seed nodes have been started it is no problem to shut down the first
seed node. If the first seed node is restarted it will first try join the other 
seed nodes in the existing cluster.

You can disable automatic joining with configuration::

      akka.cluster.auto-join = off

Then you need to join manually, using :ref:`cluster_jmx_java` or :ref:`cluster_command_line_java`.
You can join to any node in the cluster. It doesn't have to be configured as
seed node. If you are not using auto-join there is no need to configure
seed nodes at all.

Joining can also be performed programatically with ``Cluster.get(system).join(address)``.

Unsuccessful join attempts are automatically retried after the time period defined in 
configuration property ``retry-unsuccessful-join-after``. When using auto-joining with
``seed-nodes`` this means that a new seed node is picked. When joining manually or
programatically this means that the last join request is retried. Retries can be disabled by
setting the property to ``off``.

An actor system can only join a cluster once. Additional attempts will be ignored.
When it has successfully joined it must be restarted to be able to join another
cluster or to join the same cluster again. It can use the same host name and port
after the restart, but it must have been removed from the cluster before the join
request is accepted.

Automatic vs. Manual Downing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When a member is considered by the failure detector to be unreachable the
leader is not allowed to perform its duties, such as changing status of
new joining members to 'Up'. The status of the unreachable member must be
changed to 'Down'. This can be performed automatically or manually. By
default it must be done manually, using using :ref:`cluster_jmx_java` or
:ref:`cluster_command_line_java`.

It can also be performed programatically with ``Cluster.get(system).down(address)``.

You can enable automatic downing with configuration::

      akka.cluster.auto-down = on

Be aware of that using auto-down implies that two separate clusters will
automatically be formed in case of network partition. That might be
desired by some applications but not by others.

.. _cluster_subscriber_java:

Subscribe to Cluster Events
^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can subscribe to change notifications of the cluster membership by using
``Cluster.get(system).subscribe(subscriber, to)``. A snapshot of the full state,
``akka.cluster.ClusterEvent.CurrentClusterState``, is sent to the subscriber
as the first event, followed by events for incremental updates.

There are several types of change events, consult the API documentation
of classes that extends ``akka.cluster.ClusterEvent.ClusterDomainEvent``
for details about the events.

Worker Dial-in Example
----------------------

Let's take a look at an example that illustrates how workers, here named *backend*,
can detect and register to new master nodes, here named *frontend*.

The example application provides a service to transform text. When some text
is sent to one of the frontend services, it will be delegated to one of the
backend workers, which performs the transformation job, and sends the result back to
the original client. New backend nodes, as well as new frontend nodes, can be
added or removed to the cluster dynamically.

In this example the following imports are used:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/transformation/japi/TransformationBackend.java#imports

Messages:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/transformation/japi/TransformationMessages.java#messages

The backend worker that performs the transformation job:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/transformation/japi/TransformationBackend.java#backend

Note that the ``TransformationBackend`` actor subscribes to cluster events to detect new,
potential, frontend nodes, and send them a registration message so that they know
that they can use the backend worker.

The frontend that receives user jobs and delegates to one of the registered backend workers:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/transformation/japi/TransformationFrontend.java#frontend

Note that the ``TransformationFrontend`` actor watch the registered backend
to be able to remove it from its list of availble backend workers.
Death watch uses the cluster failure detector for nodes in the cluster, i.e. it detects
network failures and JVM crashes, in addition to graceful termination of watched
actor.

This example is included in ``akka-samples/akka-sample-cluster`` and you can try it by copying the 
`source <@github@/akka-samples/akka-sample-cluster>`_ to your
maven project, defined as in :ref:`cluster_simple_example_java`.
Run it by starting nodes in different terminal windows. For example, starting 2
frontend nodes and 3 backend nodes::

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.transformation.japi.TransformationFrontendMain" \
    -Dexec.args="2551"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.transformation.japi.TransformationBackendMain" \
    -Dexec.args="2552"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.transformation.japi.TransformationBackendMain"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.transformation.japi.TransformationBackendMain"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.transformation.japi.TransformationFrontendMain"

Node Roles
^^^^^^^^^^

Not all nodes of a cluster need to perform the same function: there might be one sub-set which runs the web front-end,
one which runs the data access layer and one for the number-crunching. Deployment of actors—for example by cluster-aware
routers—can take node roles into account to achieve this distribution of responsibilities.

The roles of a node is defined in the configuration property named ``akka.cluster.roles``
and it is typically defined in the start script as a system property or environment variable.

The roles of the nodes is part of the membership information in ``MemberEvent`` that you can subscribe to.

How To Startup when Cluster Size Reached
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A common use case is to start actors after the cluster has been initialized,
members have joined, and the cluster has reached a certain size. 

With a configuration option you can define required number of members
before the leader changes member status of 'Joining' members to 'Up'.

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/resources/factorial.conf#min-nr-of-members

In a similar way you can define required number of members of a certain role
before the leader changes member status of 'Joining' members to 'Up'.

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/resources/factorial.conf#role-min-nr-of-members

You can start the actors in a ``registerOnMemberUp`` callback, which will 
be invoked when the current member status is changed tp 'Up', i.e. the cluster
has at least the defined number of members.

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/factorial/japi/FactorialFrontendMain.java#registerOnUp

This callback can be used for other things than starting actors.

Cluster Singleton Pattern
^^^^^^^^^^^^^^^^^^^^^^^^^

For some use cases it is convenient and sometimes also mandatory to ensure that
you have exactly one actor of a certain type running somewhere in the cluster.

This can be implemented by subscribing to ``LeaderChanged`` or ``RoleLeaderChanged``
events, but there are several corner cases to consider. Therefore, this specific use 
case is made easily accessible by the :ref:`cluster-singleton` in the contrib module.
You can use it as is, or adjust to fit your specific needs. 

Distributed Publish Subscribe Pattern
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

See :ref:`distributed-pub-sub` in the contrib module.

Cluster Client
^^^^^^^^^^^^^^

See :ref:`cluster-client` in the contrib module.

Failure Detector
^^^^^^^^^^^^^^^^

The nodes in the cluster monitor each other by sending heartbeats to detect if a node is
unreachable from the rest of the cluster. The heartbeat arrival times is interpreted
by an implementation of
`The Phi Accrual Failure Detector <http://ddg.jaist.ac.jp/pub/HDY+04.pdf>`_.

The suspicion level of failure is given by a value called *phi*.
The basic idea of the phi failure detector is to express the value of *phi* on a scale that
is dynamically adjusted to reflect current network conditions.

The value of *phi* is calculated as::

  phi = -log10(1 - F(timeSinceLastHeartbeat))

where F is the cumulative distribution function of a normal distribution with mean
and standard deviation estimated from historical heartbeat inter-arrival times.

In the :ref:`cluster_configuration_java` you can adjust the ``akka.cluster.failure-detector.threshold``
to define when a *phi* value is considered to be a failure.

A low ``threshold`` is prone to generate many false positives but ensures
a quick detection in the event of a real crash. Conversely, a high ``threshold``
generates fewer mistakes but needs more time to detect actual crashes. The
default ``threshold`` is 8 and is appropriate for most situations. However in
cloud environments, such as Amazon EC2, the value could be increased to 12 in
order to account for network issues that sometimes occur on such platforms.

The following chart illustrates how *phi* increase with increasing time since the
previous heartbeat.

.. image:: ../images/phi1.png

Phi is calculated from the mean and standard deviation of historical
inter arrival times. The previous chart is an example for standard deviation
of 200 ms. If the heartbeats arrive with less deviation the curve becomes steeper,
i.e. it is possible to determine failure more quickly. The curve looks like this for
a standard deviation of 100 ms.

.. image:: ../images/phi2.png

To be able to survive sudden abnormalities, such as garbage collection pauses and
transient network failures the failure detector is configured with a margin,
``akka.cluster.failure-detector.acceptable-heartbeat-pause``. You may want to
adjust the :ref:`cluster_configuration_java` of this depending on you environment.
This is how the curve looks like for ``acceptable-heartbeat-pause`` configured to
3 seconds.

.. image:: ../images/phi3.png

Death watch uses the cluster failure detector for nodes in the cluster, i.e. it 
generates ``Terminated`` message from network failures and JVM crashes, in addition 
to graceful termination of watched actor. 

Cluster Aware Routers
^^^^^^^^^^^^^^^^^^^^^

All :ref:`routers <routing-java>` can be made aware of member nodes in the cluster, i.e.
deploying new routees or looking up routees on nodes in the cluster.
When a node becomes unavailble or leaves the cluster the routees of that node are
automatically unregistered from the router. When new nodes join the cluster additional
routees are added to the router, according to the configuration.

When using a router with routees looked up on the cluster member nodes, i.e. the routees
are already running, the configuration for a router looks like this:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/multi-jvm/scala/sample/cluster/stats/StatsSampleSpec.scala#router-lookup-config

It is the relative actor path defined in ``routees-path`` that identify what actor to lookup. 
It is possible to limit the lookup of routees to member nodes tagged with a certain role by
specifying ``use-role``.

``nr-of-instances`` defines total number of routees in the cluster, but there will not be
more than one per node. Setting ``nr-of-instances`` to a high value will result in new routees
added to the router when nodes join the cluster.

The same type of router could also have been defined in code:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsService.java#router-lookup-in-code

When using a router with routees created and deployed on the cluster member nodes
the configuration for a router looks like this:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/multi-jvm/scala/sample/cluster/stats/StatsSampleSingleMasterSpec.scala#router-deploy-config


It is possible to limit the deployment of routees to member nodes tagged with a certain role by
specifying ``use-role``.

``nr-of-instances`` defines total number of routees in the cluster, but the number of routees
per node, ``max-nr-of-instances-per-node``, will not be exceeded. Setting ``nr-of-instances``
to a high value will result in creating and deploying additional routees when new nodes join
the cluster.

The same type of router could also have been defined in code:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsService.java#router-deploy-in-code

See :ref:`cluster_configuration_java` section for further descriptions of the settings.


Router Example with Lookup of Routees
-------------------------------------

Let's take a look at how to use cluster aware routers.

The example application provides a service to calculate statistics for a text.
When some text is sent to the service it splits it into words, and delegates the task
to count number of characters in each word to a separate worker, a routee of a router.
The character count for each word is sent back to an aggregator that calculates
the average number of characters per word when all results have been collected.

In this example we use the following imports:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsService.java#imports

Messages:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsMessages.java#messages

The worker that counts number of characters in each word:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsWorker.java#worker

The service that receives text from users and splits it up into words, delegates to workers and aggregates:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsService.java#service

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsAggregator.java#aggregator


Note, nothing cluster specific so far, just plain actors.

We can use these actors with two different types of router setup. Either with lookup of routees,
or with create and deploy of routees. Remember, routees are the workers in this case.

We start with the router setup with lookup of routees. All nodes start ``StatsService`` and
``StatsWorker`` actors and the router is configured with ``routees-path``:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/resources/application.conf#config-router-lookup

This means that user requests can be sent to ``StatsService`` on any node and it will use
``StatsWorker`` on all nodes. There can only be one worker per node, but that worker could easily
fan out to local children if more parallelism is needed.

This example is included in ``akka-samples/akka-sample-cluster`` and you can try it by copying the 
`source <@github@/akka-samples/akka-sample-cluster>`_ to your
maven project, defined as in :ref:`cluster_simple_example_java`.
Run it by starting nodes in different terminal windows. For example, starting 3
service nodes and 1 client::

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleMain" \
    -Dexec.args="2551"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleMain" \
    -Dexec.args="2552"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleMain"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleMain"


Router Example with Remote Deployed Routees
-------------------------------------------

The above setup is nice for this example, but we will also take a look at how to use
a single master node that creates and deploys workers. To keep track of a single
master we use the :ref:`cluster-singleton` in the contrib module. The ``ClusterSingletonManager``
is started on each node.

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsSampleOneMasterMain.java#create-singleton-manager

We also need an actor on each node that keeps track of where current single master exists and
delegates jobs to the ``StatsService``.

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/stats/japi/StatsFacade.java#facade

The ``StatsFacade`` receives text from users and delegates to the current ``StatsService``, the single
master. It listens to cluster events to lookup the ``StatsService`` on the leader node. The master runs 
on the same node as the leader of the cluster members, which is nothing more than the address currently 
sorted first in the member ring, i.e. it can change when new nodes join or when current leader leaves.

All nodes start ``StatsFacade`` and the ``ClusterSingletonManager``. The router is now configured like this:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/resources/application.conf#config-router-deploy

This example is included in ``akka-samples/akka-sample-cluster`` and you can try it by copying the 
`source <@github@/akka-samples/akka-sample-cluster>`_ to your
maven project, defined as in :ref:`cluster_simple_example_java`. Also add the `akka-contrib` dependency
to your pom.xml.

Run it by starting nodes in different terminal windows. For example, starting 3
service nodes and 1 client::

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleOneMasterMain" \
    -Dexec.args="2551"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleOneMasterMain" \
    -Dexec.args="2552"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleOneMasterClientMain"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.stats.japi.StatsSampleOneMasterMain"


.. note:: The above example will be simplified when the cluster handles automatic actor partitioning.

Cluster Metrics
^^^^^^^^^^^^^^^

The member nodes of the cluster collects system health metrics and publishes that to other nodes and to 
registered subscribers. This information is primarily used for load-balancing routers.

Hyperic Sigar
-------------

The built-in metrics is gathered from JMX MBeans, and optionally you can use `Hyperic Sigar <http://www.hyperic.com/products/sigar>`_
for a wider and more accurate range of metrics compared to what can be retrieved from ordinary MBeans.
Sigar is using a native OS library. To enable usage of Sigar you need to add the directory of the native library to 
``-Djava.libarary.path=<path_of_sigar_libs>`` add the following dependency::

  <dependency>
    <groupId>org.fusesource</groupId>
    <artifactId>sigar</artifactId>
    <version>@sigarVersion@</version>
  </dependency>

Download the native Sigar libraries from `Maven Central <http://repo1.maven.org/maven2/org/fusesource/sigar/@sigarVersion@/>`_

Adaptive Load Balancing
-----------------------

The ``AdaptiveLoadBalancingRouter`` performs load balancing of messages to cluster nodes based on the cluster metrics data.
It uses random selection of routees with probabilities derived from the remaining capacity of the corresponding node.
It can be configured to use a specific MetricsSelector to produce the probabilities, a.k.a. weights:

* ``heap`` / ``HeapMetricsSelector`` - Used and max JVM heap memory. Weights based on remaining heap capacity; (max - used) / max
* ``load`` / ``SystemLoadAverageMetricsSelector`` - System load average for the past 1 minute, corresponding value can be found in ``top`` of Linux systems. The system is possibly nearing a bottleneck if the system load average is nearing number of cpus/cores. Weights based on remaining load capacity; 1 - (load / processors) 
* ``cpu`` / ``CpuMetricsSelector`` - CPU utilization in percentage, sum of User + Sys + Nice + Wait. Weights based on remaining cpu capacity; 1 - utilization
* ``mix`` / ``MixMetricsSelector`` - Combines heap, cpu and load. Weights based on mean of remaining capacity of the combined selectors.
* Any custom implementation of ``akka.cluster.routing.MetricsSelector``

The collected metrics values are smoothed with `exponential weighted moving average <http://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average>`_. In the :ref:`cluster_configuration_java` you can adjust how quickly past data is decayed compared to new data.

Let's take a look at this router in action.

In this example the following imports are used:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/factorial/japi/FactorialBackend.java#imports

The backend worker that performs the factorial calculation:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/factorial/japi/FactorialBackend.java#backend

The frontend that receives user jobs and delegates to the backends via the router:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/factorial/japi/FactorialFrontend.java#frontend


As you can see, the router is defined in the same way as other routers, and in this case it is configured as follows:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/resources/application.conf#adaptive-router

It is only router type ``adaptive`` and the ``metrics-selector`` that is specific to this router, other things work 
in the same way as other routers.

The same type of router could also have been defined in code:

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/factorial/japi/FactorialFrontend.java#router-lookup-in-code

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/factorial/japi/FactorialFrontend.java#router-deploy-in-code

This example is included in ``akka-samples/akka-sample-cluster`` and you can try it by copying the 
`source <@github@/akka-samples/akka-sample-cluster>`_ to your
maven project, defined as in :ref:`cluster_simple_example_java`.
Run it by starting nodes in different terminal windows. For example, starting 3 backend nodes and 
one frontend::

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.factorial.japi.FactorialBackendMain" \
    -Dexec.args="2551"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.factorial.japi.FactorialBackendMain" \
    -Dexec.args="2552"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.factorial.japi.FactorialBackendMain"

  mvn exec:java \
    -Dexec.mainClass="sample.cluster.factorial.japi.FactorialFrontendMain"

Press ctrl-c in the terminal window of the frontend to stop the factorial calculations.


Subscribe to Metrics Events
---------------------------

It is possible to subscribe to the metrics events directly to implement other functionality.

.. includecode:: ../../../akka-samples/akka-sample-cluster/src/main/java/sample/cluster/factorial/japi/MetricsListener.java#metrics-listener

Custom Metrics Collector
------------------------

You can plug-in your own metrics collector instead of 
``akka.cluster.SigarMetricsCollector`` or ``akka.cluster.JmxMetricsCollector``. Look at those two implementations
for inspiration. The implementation class can be defined in the :ref:`cluster_configuration_java`.

.. _cluster_jmx_java:

JMX
^^^

Information and management of the cluster is available as JMX MBeans with the root name ``akka.Cluster``.
The JMX information can be displayed with an ordinary JMX console such as JConsole or JVisualVM.

From JMX you can:

* see what members that are part of the cluster
* see status of this node
* join this node to another node in cluster
* mark any node in the cluster as down
* tell any node in the cluster to leave

Member nodes are identified by their address, in format `akka.<protocol>://<actor-system-name>@<hostname>:<port>`.

.. _cluster_command_line_java:

Command Line Management
^^^^^^^^^^^^^^^^^^^^^^^

The cluster can be managed with the script `bin/akka-cluster` provided in the
Akka distribution.

Run it without parameters to see instructions about how to use the script::

  Usage: bin/akka-cluster <node-hostname:jmx-port> <command> ...

  Supported commands are:
             join <node-url> - Sends request a JOIN node with the specified URL
            leave <node-url> - Sends a request for node with URL to LEAVE the cluster
             down <node-url> - Sends a request for marking node with URL as DOWN
               member-status - Asks the member node for its current status
                     members - Asks the cluster for addresses of current members
                 unreachable - Asks the cluster for addresses of unreachable members
              cluster-status - Asks the cluster for its current status (member ring,
                               unavailable nodes, meta data etc.)
                      leader - Asks the cluster who the current leader is
                is-singleton - Checks if the cluster is a singleton cluster (single
                               node cluster)
                is-available - Checks if the member node is available
  Where the <node-url> should be on the format of 
    'akka.<protocol>://<actor-system-name>@<hostname>:<port>'

  Examples: bin/akka-cluster localhost:9999 is-available
            bin/akka-cluster localhost:9999 join akka.tcp://MySystem@darkstar:2552
            bin/akka-cluster localhost:9999 cluster-status


To be able to use the script you must enable remote monitoring and management when starting the JVMs of the cluster nodes,
as described in `Monitoring and Management Using JMX Technology <http://docs.oracle.com/javase/6/docs/technotes/guides/management/agent.html>`_

Example of system properties to enable remote monitoring and management::

  java -Dcom.sun.management.jmxremote.port=9999 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false

.. _cluster_configuration_java:

Configuration
^^^^^^^^^^^^^

There are several configuration properties for the cluster. We refer to the following
reference file for more information:


.. literalinclude:: ../../../akka-cluster/src/main/resources/reference.conf
   :language: none

Cluster Scheduler
-----------------

It is recommended that you change the ``tick-duration`` to 33 ms or less
of the default scheduler when using cluster, if you don't need to have it
configured to a longer duration for other reasons. If you don't do this
a dedicated scheduler will be used for periodic tasks of the cluster, which
introduce the extra overhead of another thread.

::

  # shorter tick-duration of default scheduler when using cluster
  akka.scheduler.tick-duration = 33ms



