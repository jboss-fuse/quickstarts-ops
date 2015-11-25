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
