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