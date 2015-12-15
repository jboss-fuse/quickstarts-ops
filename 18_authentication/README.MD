Default authentication modules in standalone are **defined** here:  
https://github.com/jboss-fuse/karaf/blob/2.4.0.redhat-6-2-x-patch/jaas/modules/src/main/resources/OSGI-INF/blueprint/karaf-jaas-module.xml

And deployed here:

```bash
# karaf
JBossFuse:karaf@root> id org.apache.karaf.jaas.modules
25

#bash
unzip -c bundle25/version0.0/bundle.jar  **karaf-jaas-module.xml

Archive:  bundle25/version0.0/bundle.jar
  inflating: OSGI-INF/blueprint/karaf-jaas-module.xml
<?xml version="1.0" encoding="UTF-8"?>
<!--

    Licensed to the Apache
...

```

In this file you can see the definition of several `LoginModule`s, and in particular `org.apache.karaf.jaas.modules.audit.FileAuditLoginModule`

You can see them reference from Karaf CLI with:

```bash
JBossFuse:karaf@root> jaas:realms
Index Realm                Module Class
    1 karaf org.apache.karaf.jaas.modules.properties.PropertiesLoginModule
    2 karaf org.apache.karaf.jaas.modules.publickey.PublickeyLoginModule
    3 karaf org.apache.karaf.jaas.modules.audit.FileAuditLoginModule
    4 karaf org.apache.karaf.jaas.modules.audit.EventAdminAuditLoginModul
```

**In Fabric mode** `jaas` configuration is defined as an SCR component:

https://github.com/jboss-fuse/fabric8/blob/1.2.0.redhat-6-2-x-patch/fabric/fabric-jaas/src/main/java/io/fabric8/jaas/FabricJaasRealm.java

The `karaf` realm defined there has a higher rank(99) compared to the realm defined in standalone, so it overrides the old one.

You can still see reference of the *shaded* defined modules pass `-h` to `jaas:realms`.

So if you want to modify the definition of Fabric `karaf` realm, you have to deploy your configuration with a **rank higher than 99**

The following examples does so, redefining in `blueprint` syntax a reference to `ZookeeperLoginModule` and also declares a couple of additional ones.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
    Licensed to the Apache Software Foundation (ASF) under one or more
    contributor license agreements.  See the NOTICE file distributed with
    this work for additional information regarding copyright ownership.
    The ASF licenses this file to You under the Apache License, Version 2.0
    (the "License"); you may not use this file except in compliance with
    the License.  You may obtain a copy of the License at
       http://www.apache.org/licenses/LICENSE-2.0
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
-->
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
           xmlns:jaas="http://karaf.apache.org/xmlns/jaas/v1.0.0"
           xmlns:cm="http://aries.apache.org/blueprint/xmlns/blueprint-cm/v1.1.0"
           xmlns:ext="http://aries.apache.org/blueprint/xmlns/blueprint-ext/v1.0.0">

    <type-converters>
        <bean class="org.apache.karaf.jaas.modules.properties.PropertiesConverter"/>
    </type-converters>

    <!-- Allow usage of System properties, especially the karaf.base property -->
    <ext:property-placeholder placeholder-prefix="$[" placeholder-suffix="]"/>

    <!-- AdminConfig property place holder for the org.apache.karaf.jaas  -->
    <cm:property-placeholder persistent-id="io.fabric8.jaas" update-strategy="reload">
        <cm:default-properties>
            <cm:property name="encryption.name" value="basic"/>
            <cm:property name="encryption.enabled" value="true"/>
            <cm:property name="encryption.prefix" value="{CRYPT}"/>
            <cm:property name="encryption.suffix" value="{CRYPT}"/>
            <cm:property name="encryption.algorithm" value="MD5"/>
            <cm:property name="encryption.encoding" value="hexadecimal"/>
            <cm:property name="detailed.login.exception" value="false"/>
            <cm:property name="audit.file.enabled" value="true"/>
            <cm:property name="audit.file.file" value="$[karaf.data]/security/audit.log"/>
            <cm:property name="audit.eventadmin.enabled" value="true"/>
            <cm:property name="audit.eventadmin.topic" value="org/apache/karaf/login"/>
        </cm:default-properties>
    </cm:property-placeholder>

<!-- default fabric rank is 99, we need something greater than that to overwrite it it -->
    <jaas:config name="karaf" rank="100">

        <jaas:module className="io.fabric8.jaas.ZookeeperLoginModule" flags="required">
            enabled = ${audit.file.enabled}
            file = ${audit.file.file}
            encryption.name = ${encryption.name}
            encryption.enabled = ${encryption.enabled}
            encryption.prefix = ${encryption.prefix}
            encryption.suffix = ${encryption.suffix}
            encryption.algorithm = ${encryption.algorithm}
            encryption.encoding = ${encryption.encoding}
            path = /fabric/authentication/users
        </jaas:module>
        <jaas:module className="org.apache.karaf.jaas.modules.audit.FileAuditLoginModule" flags="optional">
            enabled = ${audit.file.enabled}
            file = ${audit.file.file}
        </jaas:module>
        <jaas:module className="org.apache.karaf.jaas.modules.audit.EventAdminAuditLoginModule" flags="optional">
            enabled = ${audit.eventadmin.enabled}
            topic = ${audit.eventadmin.topic}
        </jaas:module>
    </jaas:config>

</blueprint>

```
