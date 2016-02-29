# JVM options and Enviroment Variables
In JBoss Fuse you can define your JVM options and environment variables on two different levels:

- JBoss Fuse Server
- Child Container

#JBoss Fuse Server

The main JVM Options are specified inside `jboss-fuse-6.2.*/bin/setenv`

Here you can find the following environment variables exported:

```shell
JAVA_MIN_MEM=512M # Minimum memory for the JVM
JAVA_MAX_MEM=1024M # Maximum memory for the JVM
JAVA_PERM_MEM=128M # Minimum perm memory for the JVM
JAVA_MAX_PERM_MEM=256M # Maximum memory for the JVM
export JAVA_MIN_MEM
export JAVA_MAX_MEM
export JAVA_PERM_MEM
export JAVA_MAX_PERM_MEM
```

These variables are:
- `JAVA_MIN_MEM` : minimum memory for the JVM (default is 512M).
- `JAVA_MAX_MEM` : maximum memory for the JVM (default is 1024M).
- `JAVA_PERM_MEM` : minimum perm memory for the JVM (default is 128M).
- `JAVA_MAX_PERM_MEM` : maximum perm memory for the JVM (default is 256M).

There is also another environment variable that can be exported:

- `EXTRA_JAVA_OPTS` : this one will contains any other JVM Option you'd like to use (by default this is commented in bin/setenv)

For the Karaf side the enviroment variables are the following:
- `KARAF_HOME` : the location of your Apache Karaf installation (default is found depending where you launch the startup script).
- `KARAF_BASE` : the location of your Apache Karaf base (default is KARAF_HOME).
- `KARAF_DATA` : the location of your Apache Karaf data folder (default is KARAF_BASE/data).
- `KARAF_ETC` : the location of your Apache Karaf etc folder (default is KARAF_BASE/etc).
- `KARAF_OPTS` : extra arguments passed to the Java command line (default is null).
- `KARAF_REDIRECT` : enable/set the std/err redirection when using bin/start

All these variable are commented in bin/setenv

When you'll launch your Jboss Fuse instance with the default settings you will see the following options for your Java process:

```shell
-XX:PermSize=128M -XX:MaxPermSize=256M -Dcom.sun.management.jmxremote -Dkaraf.restart.jvm.supported=true -Djava.endorsed.dirs=/usr/lib/jvm/jdk1.7.0_79//jre/lib/endorsed:/usr/lib/jvm/jdk1.7.0_79//lib/endorsed:/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/lib/endorsed -Djava.ext.dirs=/usr/lib/jvm/jdk1.7.0_79//jre/lib/ext:/usr/lib/jvm/jdk1.7.0_79//lib/ext:/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/lib/ext -Dkaraf.instances=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/instances -Dkaraf.home=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084 -Dkaraf.base=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084 -Dkaraf.data=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/data -Dkaraf.etc=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/etc -Djava.io.tmpdir=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/data/tmp -Djava.util.logging.config.file=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/etc/java.util.logging.properties -Djavax.management.builder.initial=org.apache.karaf.management.boot.KarafMBeanServerBuilder -Densemble.auto.start=false -Dagent.auto.start=true -Dkaraf.startLocalConsole=true -Dkaraf.startRemoteShell=true -classpath /home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/lib/karaf-jaas-boot.jar:/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/lib/karaf.jar:/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/lib/karaf-jmx-boot.jar org.apache.karaf.main.Main
```

If you need to make adjustments to you JBoss Fuse server options you'll need to modify these variables in `bin/setenv`.

#Root container and child containers

The environment variables of the previous section are the default one for the Fuse root container and all of his childs.

Suppose you've created your child container and now you want to change the default options when it starts.

You can do this with the following command:

```shell
container-edit-jvm-options -f child
```

You will get the JVM options for your child container. The defaults are the following:

```shell
-Dcom.sun.management.jmxremote -Dorg.jboss.gravia.repository.storage.dir=data/repository -Dzookeeper.url=ovpn-112-41.phx2.redhat.com:2181 -Dzookeeper.password.encode=true -Dzookeeper.password=ZKENC=YWRtaW4= -Xmx768m -XX:MaxPermSize=256m -XX:+UnlockDiagnosticVMOptions -XX:+UnsyncloadClass -Dio.fabric8.datastore.gitRemotePollInterval=60000 -Dio.fabric8.datastore.component.name=io.fabric8.datastore -Dio.fabric8.datastore.importDir=fabric -Dio.fabric8.datastore.felix.fileinstall.filename=file:/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/etc/io.fabric8.datastore.cfg -Dio.fabric8.datastore.service.pid=io.fabric8.datastore -Densemble.auto.start=false -Dagent.auto.start=true -Djava.util.logging.config.file=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/instances/child/etc/java.util.logging.properties -Djavax.management.builder.initial=org.apache.karaf.management.boot.KarafMBeanServerBuilder -Djava.endorsed.dirs=/usr/lib/jvm/jdk1.7.0_79/jre/jre/lib/endorsed:/usr/lib/jvm/jdk1.7.0_79/jre/lib/endorsed:/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/lib/endorsed -Djava.ext.dirs=/usr/lib/jvm/jdk1.7.0_79/jre/jre/lib/ext:/usr/lib/jvm/jdk1.7.0_79/jre/lib/ext:/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/lib/ext -Dkaraf.home=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084 -Dkaraf.base=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/instances/child -Dkaraf.data=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/instances/child/data -Dkaraf.etc=/home/oscerd/playground/jboss-fuse/jboss-fuse-6.2.1.redhat-084/instances/child/etc -Dkaraf.startLocalConsole=false -Dkaraf.startRemoteShell=true
```

To edit the JVM options of the child container you can run the following command:

```shell
container-edit-jvm-options child `<jvmopts>`
```

Then stop and restart your child container.

The child default JVM options are the following:
- `-Dcom.sun.management.jmxremote` 
- `-Dorg.jboss.gravia.repository.storage.dir=data/repository`
- `-Xmx768m`
- `-XX:MaxPermSize=256m`
- `-XX:+UnlockDiagnosticVMOptions -XX:+UnsyncloadClass`
- `-Dzookeeper.url`
- `-Dzookeeper.password.encode`
- `-Dzookeeper.password`
- `-Densemble.auto.start`

Each time you run `container-edit-jvm-options` command passing a JVM options list, the options will be considered as extra options, then, if you pass some of the default options as argument, you'll get them 
duplicated in your child container.

For the `-XX:MaxPermSize` variable, in the next Fuse 6.3.0 release, we will avoid adding this option in case of JDK8 use, as that option is deprecated in that scenario.



