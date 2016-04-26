##### HTTP and HTTPS

HTTP and HTTPS ports are controlled, in Karaf and Fabric by these `pid`s:

```
org.ops4j.pax.web.cfg/org.osgi.service.http.port
org.ops4j.pax.web.cfg/org.osgi.service.http.port.secure
```

So you should always set your desired values, via ConfigAdmin or configuration, to those pids.

Be aware that you can set **additional** ports, directly in `jetty.xml` configuration.
But unless you use there an invalid value like `0` or the same exact value that is already defined to the `pid` above, you are risking to open another listening port.

This is the case with default configuration of JBoss Fuse 6.2.1 where  `jetty.xml` defines this node:

```xml
<Set name="port">
    <Property name="jetty.port" default="8181"/>
</Set>
```

Introducing the *side effect* that if you keep this value and also change `org.ops4j.pax.web.cfg/org.osgi.service.http.port` to a value different than `8181`, you will end up with Jetty listening on 2 different ports.
The suggestion is to comment out or completely remove that `<Set/>` node from you `jetty.xml` and to use only ConfigAdmin to control that value.

##### SSHD

The default port for `sshd` embedded service is `8101` and it's default value is present in both the `ssh` feature definition and in an explicit OSGi config file:

```
etc/org.apache.karaf.shell.cfg
```

Either change it directly in the file, or with usual ConfigAdmin operations (that will persist it on the same file, anyway.)

##### RMI port

RMI port is defined in this file `etc/org.apache.karaf.management.cfg`. Notice that the `rmiRegistryPort` variable is used in the composite `serviceUrl` variable, so be careful regarding which one you modify.

##### Fabric --min-port and --max-port

`--minPort` and `--max-port` are parameters that you can specify at `fabric:create` and `fabric:join` time to influence the ports chosed by the managed services running on your container.  

NOTE:
The range is going to be used for the new services that will be created and deployed on the container, like Zookeeper for examples and others.  
But it **will not** affect already running services, that came from the Standalone instance. Most notably: http/https, jmx, ssh. To change this "pre-existing" assignment you still have to pass through usual OSGi-ConfigAdmin options.

See also: https://access.redhat.com/solutions/1256553
