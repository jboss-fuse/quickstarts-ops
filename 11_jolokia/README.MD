- Full example based on **Nagios**, **check_jmx** and **Jolokia** : http://giallone.blogspot.it/2014/01/monitoring-jboss-fuse-esb-with-nagios.html

#### sample `curl` invocations:

##### GET wtih JSON payload - invoke operation
```bash
curl -X GET -u admin:admin -d "{'config': {}, 'mbean': 'java.lang:type=OperatingSystem', 'type': 'read'}"  "http://localhost:8181/jolokia"
```

##### GET with inlined params - read
```bash
curl -u admin:admin "http://localhost:8181/jolokia/read/java.lang:type=Memory/HeapMemoryUsage"
```

##### GET with inlined params - exec
```bash
curl -u admin:admin "http://localhost:8181/jolokia/exec/jolokia:type=ServerHandler,qualifier=hawtio/mBeanServersInfo"
```

##### GET with  params urlencoded and separated by slashes
```bash
curl -X GET -u admin:admin "http://localhost:8181/jolokia/exec/io.fabric8:type=Fabric/changeCreateOptionsField/remotessh2/jvmOpts/-XX%3ANewRatio%3D7"
```

##### Script friendly:

```bash
# params
_HOST=localhost:8181
_CONTAINERID=remotessh2
_JVM_OPTS=-XX:NewRatio=6%20-Xmx512m # use %20 instead of SPACE

# invocation
curl -X GET -u admin:admin "http://$_HOST/jolokia/exec/io.fabric8:type=Fabric/changeCreateOptionsField/$_CONTAINERID/jvmOpts/$_JVM_OPTS"
```

##### Modify Jolokia base config:
*From 6.3*
Jolokia agent is deployed automatically with `io.hawt.web.JolokiaConfiguredAgentServlet` that extends Jolokia native `org.jolokia.http.AgentServlet` class, defined in `hawtio-web/WEB-INF/web.xml`.
To customize Jolokia Servlet configuration, according to the parameters that it supports and that are defined here:
https://jolokia.org/reference/html/agents.html#agent-war-init-params

You have to pass them as java system properties, prefixed with `jolokia.`.

EX.
```
-Djolokia.policyLocation=file:///home/fuse/my-access.xml
```

**NOTE** The parameter `restrictorClass` is already used in `hawtio-web/WEB-INF/web.xml` to implement role-based access control (RBAC) for Jolokia invocations. If you want to use your own Jolokia restrictor, make sure to extend `io.hawt.web.RBACRestrictor` to implement your own restrictor class. Otherwise, part of Hawtio's RBAC functions will be lost.

##### Log all operations performed server side
See https://github.com/rhuss/jolokia/issues/113#issuecomment-21931414
In 6.2 it's not possible to automatically print to standard out without alterting `WEB.xml` defined in `hawtio-web`.
So logs can be manually collected via JMX `jolokia:type=Config` setting the attribute `Debug` to `true` and manually invoking operation `debugInfo`.
In 6.3, using the above approach allows to alter the default at startup time.

#### RBAC
By default, every access to Jolokia on Fuse is protected by RBAC (role-based access control) based on JMX ACL configuration files under `etc/auth/`.

For example, suppose you have an user `"viewer"` with the `viewer` role whose password is `"viewer"`. The following invocation to Jolokia will fail with the status code `403`:

```bash
$ curl -u viewer:viewer "http://localhost:8181/jolokia/exec/java.lang:type=Memory/gc()"
{"request":{"mbean":"java.lang:type=Memory","type":"exec","operation":"gc()"},"error_type":"java.lang.Exception","error":"java.lang.Exception : Operation gc() forbidden for MBean java.lang:type=Memory","status":403}
```

This is because the `etc/auth/jmx.acl.java.lang.Memory.cfg` file has the following ACL:
```
gc = Deployer,Auditor,Administrator,SuperUser,admin
```
and invocation to `gc()` operation of `java.lang:type=Memory` is thus forbidden.

For more information on RBAC, refer to [1.5 RBAC examples](../05_rbac/README.MD).
