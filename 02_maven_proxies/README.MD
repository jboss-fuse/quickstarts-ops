[Official documentation about maven proxies](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html-single/Configuring_and_Running_JBoss_Fuse/index.html#FESBFabricMavenProxyConfig)

JBoss Fuse uses many different properties to configure Maven repositories. This is to give flexibility to each single component.  The final result might look confusing to the use that is not sure which property it has to configure, but with a brief recap, it's easier to not get lost:

##### Standalone
- `org.ops4j.pax.url.mvn/org.ops4j.pax.url.mvn.repositories` is the list of repositories used while in **Standalone mode** and used by `install` and `featues:install` commands.

###### non profile commands in Fabric mode
- `io.fabric8.maven/io.fabric8.maven.repositories` controls the list of repositories used by `install` in **Fabric mode**
- `io.fabric8.maven/io.fabric8.maven.settings` is used to specify the location of a Maven `settings.xml` file where you might want to define proxies, mirrors, servers and repositories.

##### profiles commands in Fabric mode
- `io.fabric8.agent/org.ops4j.pax.url.mvn.repositories` is used for everything that leverages the Deployment Agent, so everything that has to do with Fabric Profiles
- `io.fabric8.maven/io.fabric8.maven.settings` is used to specify the location of a Maven `settings.xml` file where you might want to define proxies, mirrors, servers and repositories.
- `io.fabric8.maven/io.fabric8.maven.defaultRepositories` is used for local repositories (nothing over http), and it's a good idea to configure it with `file:\${runtime.home}/\${karaf.default.repository}@snapshots@id=karaf-default,file:\${runtime.data}/maven/upload@snapshots@id=fabric-upload` so that you have access to files uploaded with patches
- In case of child containers, a deadlock situation can happen between root and children containers both trying to use the use repo. In those case a suggestion is to dedicate a private Maven repo for each container with this configuration:
```
fabric:profile-edit --pid io.fabric8.maven/io.fabric8.maven.localRepository='${karaf.data}/repository' default
```


#### Notes:

- if you have **repositories** defined in a `settings.xml` file, to make them read by Maven, you need to **prepend a plus symbol** in front of the list of your explicitely defined repositories.

ex.
```bash
profile-edit --pid io.fabric8.agent/org.ops4j.pax.url.mvn.settings='file:///opt/rh/s.xml' default
# the leading "plus" symbol is required, otherwise repos defined in s.xml won't be read
profile-edit --pid io.fabric8.agent/org.ops4j.pax.url.mvn.repositories='+http://repo1.maven.org/maven2@id=maven.central.repo'  default
```

Sample configuration:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd" xmlns="http://maven.apache.org/SETTINGS/1.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <proxies>
     <proxy>
        <id>example-proxy</id>
        <active>true</active>
        <protocol>http</protocol>
        <host>172.17.0.4</host>
        <port>3128</port>
      </proxy>
    </proxies>
    <profiles>
        <profile>
            <id>fuse-repos</id>
            <activation>
               <activeByDefault>true</activeByDefault>
             </activation>
            <repositories>
                <repository>
                    <id>fuse-qe-repo</id>
                    <url>http://172.17.0.3:8081/nexus/content/repositories/releases</url>
                    <layout>default</layout>
                </repository>
            </repositories>
        </profile>
    </profiles>
    <activeProfiles>
        <activeProfile>fuse-repos</activeProfile>
    </activeProfiles>
</settings>
```

##### log levels
- useful loggers that can be turned more verbose(`TRACE`) are:
```
io.fabric8.fabric-maven - INFO
io.fabric8.fabric-maven-proxy - INFO
io.fabric8.maven.proxy.impl.MavenProxyRegistrationHandler - INFO
io.fabric8.maven.proxy.impl.MavenProxyServletSupport - INFO
io.fabric8.maven.proxy.impl.MavenSecureHttpContext - INFO
io.fabric8.maven.url.internal.Activator - INFO
io.fabric8.maven.url.internal.AetherBasedResolver - INFO
io.fabric8.maven.url.internal.Connection - INFO
io.fabric8.maven.util.MavenConfigurationImpl - INFO
io.fabric8.maven.util.MavenRepositoryURL - INFO
org.apache.maven.repository.internal.DefaultArtifactDescriptorReader - INFO
org.apache.maven.repository.internal.DefaultVersionRangeResolver - INFO
org.apache.maven.repository.internal.DefaultVersionResolver - INFO
org.eclipse.aether.connector.basic.BasicRepositoryConnector - INFO
org.eclipse.aether.internal.impl.DefaultArtifactResolver - INFO
org.eclipse.aether.internal.impl.DefaultDependencyCollector - INFO
org.eclipse.aether.internal.impl.DefaultDeployer - INFO
org.eclipse.aether.internal.impl.DefaultInstaller - INFO
org.eclipse.aether.internal.impl.DefaultLocalRepositoryProvider - INFO
org.eclipse.aether.internal.impl.DefaultMetadataResolver - INFO
org.eclipse.aether.internal.impl.DefaultOfflineController - INFO
org.eclipse.aether.internal.impl.DefaultRemoteRepositoryManager - INFO
org.eclipse.aether.internal.impl.DefaultRepositoryConnectorProvider - INFO
org.eclipse.aether.internal.impl.DefaultRepositoryEventDispatcher - INFO
org.eclipse.aether.internal.impl.DefaultRepositoryLayoutProvider - INFO
org.eclipse.aether.internal.impl.DefaultRepositorySystem - INFO
org.eclipse.aether.internal.impl.DefaultTransporterProvider - INFO
org.eclipse.aether.internal.impl.DefaultUpdateCheckManager - INFO
org.eclipse.aether.internal.impl.DefaultUpdatePolicyAnalyzer - INFO
org.eclipse.aether.internal.impl.EnhancedLocalRepositoryManager - INFO
org.eclipse.aether.internal.impl.SimpleLocalRepositoryManager - INFO
org.eclipse.aether.internal.impl.WarnChecksumPolicy - INFO
org.eclipse.aether.transport.wagon.WagonTransporter - INFO
```


##### validation
- a part of the specified configuration, you should verify that you have connectivity across nodes and that you client nodes can access repos with or without proxies.  
Sample invocations are:
```bash
wget http://172.17.0.2:8181/maven/download/io/fabric8/fabric8-karaf/1.2.0.redhat-621081/fabric8-karaf-1.2.0.redhat-621081.zip
```
