# Insight

Fabric mode, offers a logging and monitoring functionality called Insight.

Insight is able to aggregate logs from multiple sources and to store them in a single searchable location to allow metrics and business analysis.

Insight is powered by an instance of ElasticSearch in combination with and set of Hawt.io custom views, that allow to:
- see a custom page, designed around Camel Exchanges
- see a generic embedded instance of Kibana, an open source front end for Elastic Search with powerful capabilities.


##### Install Insight:

You need to deploy several profiles. The reason why there are multiple of them it's because you might want a mixed topology, without each component on each node.

This is a list of profiles required:

- `insight-elasticsearch.datastore` installs an ElasticSearch instance
- `insight-console` installs Hawt.io main integration
- `insight-logs.elasticsearch` needs to be installed on each node where you want your normal logs forwarde to your centralized ElasticSearch
- `insight-metrics.elasticsearch` feeds an index with generic metrics, like jvm stats
- `insight-camel` feeds an index with metrics specifically extracted from Camel routes

Once you have deployed the relevant profiles, for a single node it would be something like:

```
container-add-profile root insight-elasticsearch.datastore insight-console insight-logs.elasticsearch insight-metrics.elasticsearch insight-camel
```

Now, when you log in Hawt.io you will find a new enty in top left menu, called Insight, that brings you to the Insight perspective.

Here you can find:

- `Camel` a search page to query for specific Exchange ids
- `Logs` tab, showing aggregated logs from all nodes
- `Camel Events` a specialized view with Camel metrics
- `Elastic Search`, an embedded instance of Kibana to build your own queries and inspect ElasticSearch indexes 

##### Install Insight for ActiveMQ

As of JBoss Fuse 6.3.0, `insight-activemq` features and profiles are not provided with the distro, but they are available for manual installation.

See https://issues.jboss.org/browse/ENTESB-6018

Required steps:

1) add the missing feature:
```
cd /opt/rh/jboss-fuse-6.2.1.redhat-159/
# add feature definition
sed -i 's#</features>#<feature name="insight-activemq" version="1.2.0.redhat-621159" resolver="(obr)"> <feature version="1.2.0.redhat-621159">mq-fabric</feature> <feature version="5.11.0.redhat-621159">activemq-client</feature> <feature>insight-elasticsearch-log-storage</feature> <bundle dependency="true">mvn:org.mvel/mvel2/2.2.6.Final</bundle> <bundle>mvn:org.codehaus.jackson/jackson-mapper-asl/1.9.12</bundle> <bundle>mvn:org.codehaus.jackson/jackson-core-asl/1.9.12</bundle> <bundle>mvn:io.fabric8.insight/insight-activemq/1.2.0.redhat-621159</bundle> </feature></features>#' system/io/fabric8/fabric8-karaf/1.2.0.redhat-621159/fabric8-karaf-1.2.0.redhat-621159-features.xml
```

2) create a profile that includes feature `insight-activemq`
```
profile-create insight-activemq
profile-edit --feature insight-activemq insight-activemq
``` 

3) Modify `broker.xml` configuration:
```
profile-edit --resource broker.xml mq-amq
# manually add this value `<bean xmlns="http://www.springframework.org/schema/beans" id="InsightActiveMQ" class="io.fabric8.insight.activemq.InsightPlugin"></bean>` to node `<plugins>`

4) Enable plugin per destination:

```
# enable plugin per destination
profile-edit --pid io.fabric8.insight.activemq/auditor.enabled=true mq-broker-default.broker_profile01
profile-edit --pid io.fabric8.insight.activemq/auditor.dest.queue://BMDA=true mq-broker-default.broker_profile01
```

Now, if you deploy usual insight profiles, and the newly created one, you will have special metrics emitted for each messages that passes through ActiveMQ, forwared to the centralized ElasticSearch index that you can query with custom Kibana queries.

Notice that there is not a specialized tab in Hawtio for ActiveMQ only metrics.


##### Use an external instance of Kibana.

You may want to avoid using Hawtio embedded Kibana version and prefer to use a standalone one.

To set up this you need necessarely a version of `3.x` of Kibana, since version `4.x` won't work.
The setup has been tested succesfully with version `3.1.3`. https://www.elastic.co/downloads/past-releases/kibana-3-1-3

Version `3.x` of Kibana is a HTML5+javascript application, that comes without a web server.
So you need to deploy it into one.

For testing purposes you can spawn a temporary one with this command, run from the Kibana root folder:
```
python -m SimpleHTTPServer 8888
```

You can now access Kibana interface at this url: `http://localhost:8888`

NOTE:
Kibana `3.x` doesn't offer much configuration capabilities, and it assumes ElasticSearch is available at this endpoint: `localhost:9200`.
In case you are running Kibana from a different node than the one you deployed Insight onto, you need to forward the remote `9200` port and re-expose it as it was local.
You can use either `ssh` or `socat` to obtain that effect.