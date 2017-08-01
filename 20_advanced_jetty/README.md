# Configure an additional handler on 6.2.1

Add the following node insie the `<Configure>` node in your `jetty.xml`
```xml
<Get name="handler">
    <Call name="addHandler">
      <Arg>
        <New id="IPAccessHandler" class="org.eclipse.jetty.server.handler.IPAccessHandler">
            <Set name="white">
             <Array type="String">
               <Item>127.0.0.1</Item>
               <Item>127.0.0.2/*.html</Item>
             </Array>
            </Set> 
            <Set name="black">
             <Array type="String">
               <Item>127.0.0.1/blacklisted</Item>
               <Item>127.0.0.2/black.html</Item>
             </Array>
            </Set>
        </New>
      </Arg>
    </Call>
</Get>

```

# Expose a specific web app on specific port (tested on 6.3).

1. Define a new connector in your `jetty.xml`

```xml
<!-- =========================================================== -->
<!-- Special server connectors -->
<!-- =========================================================== -->
<!-- This is a sample for alternative connectors, enable if needed -->
<!-- =========================================================== -->
<!--    -->
<Call name="addConnector">
    <Arg>
        <New class="org.eclipse.jetty.server.ServerConnector">
            <Arg name="server">
                <Ref refid="Server" />
            </Arg>
            <Arg name="factories">
                <Array type="org.eclipse.jetty.server.ConnectionFactory">
                    <Item>
                        <New class="org.eclipse.jetty.server.HttpConnectionFactory">
                            <Arg name="config">
                                <Ref refid="httpConfig" />
                            </Arg>
                        </New>
                    </Item>
                </Array>
            </Arg>
            <Set name="host">
                <Property name="jetty.host" default="localhost" />
            </Set>
            <Set name="port">
                <Property name="jetty.port" default="8282" />
            </Set>
            <Set name="idleTimeout">
                <Property name="http.timeout" default="30000" />
            </Set>
            <!-- IMPORTANT: set a name for the connector -->
            <Set name="name">jettyConn1</Set>
        </New>
    </Arg>
</Call>
```

2. Add a new `jetty-web.xml` deployment descriptor to your `.war` archive, inside `WEB-INF` folder:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">

<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="virtualHosts">
    <Array type="java.lang.String">
      <!-- use this syntax to bind to a specific named connector -->
      <Item>@jettyConn1</Item>
    </Array>
  </Set>
</Configure>
```

With this configuration, the `.war` containing that specific `WEB-INF\jetty-web.xml` configuration file, will be served **ONLY** by the connector on port `8282`.

See more details here: http://www.eclipse.org/jetty/documentation/current/configuring-virtual-hosts.html

An alternative approach, to avoid modifying the original jar is to use a Fragment bundle.
This is an example of how to use a file system folder to be used as a Fragment bundle:

```
mkdir -p f/WEB-INF

cat > /opt/rh/jboss-fuse-6.3.0.redhat-224/f/WEB-INF/jetty-web.xml <<"EOF"
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure_9_3.dtd">

<Configure class="org.eclipse.jetty.webapp.WebAppContext">
  <Set name="virtualHosts">
    <Array type="java.lang.String">
      <!-- use this syntax to bind to a specific named connector -->
      <Item>@jettyConn1</Item>
    </Array>
  </Set>
</Configure>
EOF

#in karaf
install 'wrap:jardir:/opt/rh/jboss-fuse-6.3.0.redhat-224/f$Bundle-SymbolicName=jetty_web&Bundle-Version=0.0.0&Fragment-Host=io.hawt.hawtio-web'
update io.hawt.hawtio-web

And now hawtio is accessible only on port `8282` (mind that it's bound only to `localhost` unless you tweak `jetty.xml` a little more)

```

# Enabling NCSA Log request in Jetty on JBoss Fuse 6.3

A common requirement is to have a separate log of all the http calls your application receive in [NCSA format](https://en.wikipedia.org/wiki/Common_Log_Format). In order to enable it in the jetty instance shipped with JBoss Fuse 6.3 we need to register an `org.eclipse.jetty.server.Handler` OSGi service. The easiest way is with the following blueprint snippet:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
           xmlns:jaas="http://karaf.apache.org/xmlns/jaas/v1.0.0"
           xmlns:httpj="http://cxf.apache.org/transports/http-jetty/configuration"
           xmlns:ext="http://aries.apache.org/blueprint/xmlns/blueprint-ext/v1.0.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://cxf.apache.org/transports/http/configuration
            http://cxf.apache.org/schemas/configuration/http-conf.xsd">

            <service ref="requestLog" interface="org.eclipse.jetty.server.Handler"/>

            <bean id="requestLog" class="org.eclipse.jetty.server.handler.RequestLogHandler">
              <property name="requestLog"  ref="requestLogImpl"/>
            </bean>

            <bean id="requestLogImpl" class="org.eclipse.jetty.server.NCSARequestLog">
              <argument value="/home/fuse/yyyy_MM_dd.log"/>
              <property name="retainDays" value="90"/>
              <property name="append" value="true"/>
              <property name="extended" value="false"/>
              <property name="filenameDateFormat" value="yyyy_MM_dd"/>
              <property name="ignorePaths">
                <array>
                  <value>/hawtio/jolokia/*</value>
                  <value>/jolokia/*</value>
                </array>
              </property>
            </bean>

</blueprint>
```

Let's explain the configuration options available:
- `<bean id="requestLogImpl" class="org.eclipse.jetty.server.NCSARequestLog">` this is the class implementing the logging, an async version of it exist called `org.eclipse.jetty.server.AsyncNCSARequestLog`.
- `<argument value="/home/fuse/yyyy_MM_dd.log"/>` here you configure where to put the logs, if you have a date patter that should match the one in `filenameDateFormat` (see afterwards) and that would be automatically substituted with the appropriate date and times.
- `<property name="retainDays" value="90"/>` configures number of days to retain the logs.
- `<property name="append" value="true"/>` if `false` the logs will be deleted at each restart of the container.
- `<property name="extended" value="false"/>` it `true` will log the extended informations about the http request.
- `<property name="filenameDateFormat" value="yyyy_MM_dd"/>` date time pattern to be used in log filename (see above).
- `<property name="ignorePaths">` here you can configure which request pattern to ignore in the log; usually you want to avoid logging for `<value>/hawtio/jolokia/*</value>` hawtio admin interface  and `<value>/jolokia/*</value>` jolokia monitoring connector as well.

## How to install it in standalone mode
To install it in standalone mode just copy the snippet in a file, let's say `ncsaLogging.xml`, tweak it to your needs and install it with:
```bash
> osgi:intall -s blueprint:file:///path/to/file/ncsaLogging.xml
```

## How to install it in fabric mode
To install it in fabric mode you need to create in an profile (for example `myProfile`), a file with the snippet's content, let's say `ncsaLogging.xml`, and then add a bundle with the following command:
```bash
> profile-edit --bundle blueprint:profile:ncsaLogging.xml myProfile
```
