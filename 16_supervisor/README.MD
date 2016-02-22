
## Fuse JON management

#### Basic introduction to JON

JON is enterprise management solution that can be used for monitoring, managing, alerting, configuration and operations with JBoss middleware products. JON is based on RHQ, so the [documentation of RHQ](http://rhq-project.github.io/rhq/ "RHQ documentation") can be also useful

The JON architecrure:

![alt text](http://lh5.ggpht.com/-gUSfosNNIYY/UclWbKOT0cI/AAAAAAAABHk/PPdRZ-a6Yns/Architecture_overview_cassandra_color.png?imgmax=800 "JON architecture")

The main components are JON server, which is the central management point. Further components are JON agents, which are installed on every host, we want to be managed by JON server. The main components, used by agents for monitoring resources on hosts are called JON plugins. Some plugins are distributed directly with JON distribution, but some have to be added to the plugin repository.
This is also the case of plugins used for monitoring of JBoss Fuse. More information to this are provided in next sections.

#### JON plugin pack for JBoss Fuse

JON plugin pack for JBoss Fuse can be downloaded from Red Hat [customer portal](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?product=jon.fuse&downloadType=distributions "Red Hat customer portal").

It is usefull to look inside the [official Fuse JON plugin pack documentation](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html-single/JON_Performance_Metrics_Reference/index.html "Fuse JON plugin documentation").


This plugin pack is used for **JBoss Fuse, JBoss A-MQ** and also **Fuse on EAP** (from version 6.2.1.)

The JBoss Fuse JON plugin pack for JBoss Fuse and Fuse on EAP version 6.2.1. contains following plugins:

* activemq-plugin
* camel-plugin
* cxf-plugin
* fuse-jboss-as-7-plugin
* fabric-groups-plugin
* fabric-plugin
* jboss-fuse-plugin
* servicemix3-plugin
* servicemix4-plugin
* switchyard-rhq-plugin
* rhq-jboss-as-7-plugin
* rhq-serverplugin-wfly-patch-bundle


#### Prerequisites for management using JON

As mentioned above in the section about the JON architecture, first of all we of course need to install JON server. We can follow the official [installation instructions](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Operations_Network/3.3/html/Installation_Guide/index.html "JON installation guide").
After successful installation we can download the JON agent from our JON server. It is located in ``Administration -> Downloads -> Agent download``. This is a single .jar file, which we then install on hosts we want to manage using:

``java -jar ${agent_jar_location} install``

We insert all information, that is required from us. The agent should be discovered in few minutes in the JON discovery queue.

The plugin pack can be simply unzipped into agent plugins directory, or it can be installed using the JON UI. Both ways will need some time to synchronize the plugin packs accross all the agents, connected to the central JON server. Of course we have to make sure that our firewall settings allow the connection on required port.

After this step, the automatic resource discovery should discover all running Fuse instances on every host. The resources can then be imported into JON. For every server, if the user name/password is different than admin/admin, we have to specify this user in the ``Server Inventory -> Connection Settings``. After this it will take few moments and all server resources should be available.

#### Fuse resources monitored by JON  

**Fuse Server resources** we see after the discovery are:

* Brokers
* Bus
* Camel contexts
* CXF endpoints
* Karaf admins
* Karaf features
* OSGI Core Bundle state
* OSGI Core Frameworks
* OSGI Core Package State
* OSGI Core Service States

**Metrics** are collected mainly for Camel contexts deployed on JBoss Fuse and for A-MQ queues, topics. There we can see for Queue/topic instances:

* Enqueued/dequeued count
* Consumer count
* Enqueue/dequeue count per minute
* Memory usage

For camel contexts:
* endpoints availability/unavailability
* Camel processors exchanges total/failed/Completed
* Camel processors Max/Min processing time
* Camel routes Max/min/last processing time
* Camel routes Exchanges failed/total/completed

**Operations**, we can use are mosly also for A-MQ queues and topics. It is possible to:
  * add/remove topic/queue
  * disable/enable/reset statistics
  * reset statistics
  * invoke GC
  * send/browse msg in topics
  * send/browse msg in queue
  * purge queue
  * get connection count
  * start/stop connection
  * enable/disable/reset connections statistics
  * check buffered messages

Camel
  * camel project deployment/undeployment/discovery
  * start/stop/restart camel route or processor

CXF
  * cxf project deploy/undeploy/discovery
  * start/stop CXF endpoints

#### Alerting with JON

There can be alerts specified in JON, that fire every time, some specified situation happens. The complete documentation can be found in the [JON documentation](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Operations_Network/3.3/html/Users_Guide/alerts-chapter.html)
