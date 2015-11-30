Official documentation:
[https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2/html-single/Configuring_and_Running_JBoss_Fuse/index.html#FESBFabricMavenProxyConfig]

JBoss Fuse uses many different properties to configure Maven repositories. This is to give flexibility to each single component.  The final result might look confusing to the use that is not sure which property it has to configure, but with a brief recap, it's easier to not get lost:

##### Standalone
- `org.ops4j.pax.url.mvn/org.ops4j.pax.url.mvn.repositories` is the list of repositories used while in **Standalone mode** and used by `install` and `featues:install` commands.

###### non profile commands in Fabric mode
- `io.fabric8.maven/io.fabric8.maven.repositories` controls the list of repositories used by `install` in **Fabric mode**
- `io.fabric8.maven/io.fabric8.maven.settings` is used to specify the location of a Maven `settings.xml` file where you might want to define proxies, mirrors, servers and repositories.

##### profiles commands in Fabric mode
- `io.fabric8.agent/org.ops4j.pax.url.mvn.repositories` is used for everything that leverages the Deployment Agent, so everything that has to do with Fabric Profiles
- `io.fabric8.maven/io.fabric8.maven.settings` is used to specify the location of a Maven `settings.xml` file where you might want to define proxies, mirrors, servers and repositories.

Notes:
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
