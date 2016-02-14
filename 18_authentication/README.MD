# Authentication
Authentication in JBoss Fuse is handled by the `jaas` system, thus configured through so called realms. In each realm `LoginModules` are used to provide and configure different means of managing Authentication sources (i.e. from `file`, from `zookeeper`, from `ldap` and from `jdbc`).

There are differences on how to change the configuration between **standalone and fabric mode** as well as on how the defaults are configured.  

## Default configuration in standalone mode
Default authentication modules in **standalone mode** are defined here in a blueprint file called karaf-jaas-module.xml ( an example can be found here:
https://github.com/jboss-fuse/karaf/blob/2.4.0.redhat-6-2-x-patch/jaas/modules/src/main/resources/OSGI-INF/blueprint/karaf-jaas-module.xml )

To find out in which bundle this blueprint is deployed do:
```bash
# from karaf console
JBossFuse:karaf@root> id org.apache.karaf.jaas.modules
25
```

In this example the blueprint mentioned before is deployed as bundle number 25 (the number might vary between installations). To see what is the actual content of the installed bundle do:
```bash
#bash with FUSE_HOME being the JBoss Fuse's installation folder
unzip -c $FUSE_HOME/data/cache/bundle25/version0.0/bundle.jar  **karaf-jaas-module.xml

Archive:  bundle25/version0.0/bundle.jar
  inflating: OSGI-INF/blueprint/karaf-jaas-module.xml
<?xml version="1.0" encoding="UTF-8"?>
<!--

    Licensed to the Apache
...
```

In this file you can see the definition of several `LoginModule`s, and in particular `org.apache.karaf.jaas.modules.audit.FileAuditLoginModule`; you can see them reference from Karaf CLI with:
```bash
# from karaf console
JBossFuse:karaf@root> jaas:realms
Index Realm                Module Class
    1 karaf org.apache.karaf.jaas.modules.properties.PropertiesLoginModule
    2 karaf org.apache.karaf.jaas.modules.publickey.PublickeyLoginModule
    3 karaf org.apache.karaf.jaas.modules.audit.FileAuditLoginModule
    4 karaf org.apache.karaf.jaas.modules.audit.EventAdminAuditLoginModul
```

## Default configuration in fabric mode
In **fabric mode** `jaas` configuration is defined as an SCR component ( an example can be found here: https://github.com/jboss-fuse/fabric8/blob/1.2.0.redhat-6-2-x-patch/fabric/fabric-jaas/src/main/java/io/fabric8/jaas/FabricJaasRealm.java).

The `karaf` realm defined there has a higher rank(99) compared to the realm defined in standalone, so it overrides the old one. It also need to be defined in a profile from which every other profile derives to be sure that all the containers apply it, usually the default profile.

You can still see reference of the *shaded* defined modules pass `-h` to `jaas:realms`.

So if you want to modify the definition of Fabric `karaf` realm, you have to deploy your configuration with a **rank higher than 99**. The following examples does so, redefining in `blueprint` syntax a reference to `ZookeeperLoginModule` and also declares a couple of additional ones.
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

## LDAP LoginModule
In order to configure JBoss Fuse authentication with ldap (i.e. authenticating against an ldap server containing USERS and ROLES) follow these steps:
1. Install the `LdapLoginModule`:
```bash
# from karaf console
JBossFuse:karaf@root> install blueprint:file://path/to/ldapjaas.xml
```
Example content of `ldapjaas.xml` would be:
```xml
<?xml version="1.0" encoding="UTF-8"?>
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
    <cm:property-placeholder persistent-id="org.apache.karaf.jaas" update-strategy="reload">
        <cm:default-properties>
            <cm:property name="detailed.login.exception" value="false"/>
            <cm:property name="encryption.name" value=""/>
            <cm:property name="encryption.enabled" value="false"/>
            <cm:property name="encryption.prefix" value="{CRYPT}"/>
            <cm:property name="encryption.suffix" value="{CRYPT}"/>
            <cm:property name="encryption.algorithm" value="MD5"/>
            <cm:property name="encryption.encoding" value="hexadecimal"/>
            <cm:property name="audit.file.enabled" value="true"/>
            <cm:property name="audit.file.file" value="$[karaf.data]/security/audit.log"/>
            <cm:property name="audit.eventadmin.enabled" value="true"/>
            <cm:property name="audit.eventadmin.topic" value="org/apache/karaf/login"/>
        </cm:default-properties>
    </cm:property-placeholder>

    <jaas:config name="karaf" rank="200">
      <jaas:module className="org.apache.karaf.jaas.modules.ldap.LDAPLoginModule" flags="sufficient">
        initialContextFactory=com.sun.jndi.ldap.LdapCtxFactory
        connection.username=cn=admin,dc=example,dc=org
        connection.password=${password}
        connection.protocol=
        connection.url=ldap://${ldaphostserver}:${ldapport}
        user.base.dn=ou=People,dc=example,dc=org
        user.filter=(uid=%u)
        user.search.subtree=true
        role.base.dn=ou=Groups,dc=example,dc=org
        role.name.attribute=cn
        role.filter=(memberuid=%u)
        role.search.subtree=true
        authentication=simple
      </jaas:module>
	        <jaas:module className="org.apache.karaf.jaas.modules.properties.PropertiesLoginModule" flags="optional">
            users = $[karaf.etc]/users.properties
            detailed.login.exception = ${detailed.login.exception}
            encryption.name = ${encryption.name}
            encryption.enabled = ${encryption.enabled}
            encryption.prefix = ${encryption.prefix}
            encryption.suffix = ${encryption.suffix}
            encryption.algorithm = ${encryption.algorithm}
            encryption.encoding = ${encryption.encoding}
        </jaas:module>
        <jaas:module className="org.apache.karaf.jaas.modules.publickey.PublickeyLoginModule" flags="optional">
            users = $[karaf.etc]/keys.properties
            detailed.login.exception = ${detailed.login.exception}
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

    <!-- The Backing Engine Factory Service for the JDBCLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.jdbc.JDBCBackingEngineFactory"/>
    </service>

    <!-- AutoEncryption support -->
    <bean class="org.apache.karaf.jaas.modules.properties.AutoEncryptionSupport"
            init-method="init" destroy-method="destroy">
        <argument>
            <map>
                <entry key="org.osgi.framework.BundleContext" value-ref="blueprintBundleContext"/>
                <entry key="users" value="$[karaf.etc]/users.properties"/>
                <entry key="encryption.name" value="${encryption.name}"/>
                <entry key="encryption.enabled" value="${encryption.enabled}"/>
                <entry key="encryption.prefix" value="${encryption.prefix}"/>
                <entry key="encryption.suffix" value="${encryption.suffix}"/>
                <entry key="encryption.algorithm" value="${encryption.algorithm}"/>
                <entry key="encryption.encoding" value="${encryption.encoding}"/>
            </map>
        </argument>
    </bean>

    <!-- The Backing Engine Factory Service for the PropertiesLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.properties.PropertiesBackingEngineFactory"/>
    </service>

    <!-- The Backing Engine Factory Service for the PublickeyLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.publickey.PublickeyBackingEngineFactory"/>
    </service>

    <service interface="org.apache.karaf.jaas.modules.EncryptionService" ranking="-1">
        <service-properties>
            <entry key="name" value="basic"/>
        </service-properties>
        <bean class="org.apache.karaf.jaas.modules.encryption.BasicEncryptionService"/>
    </service>

</blueprint>
```
There are also the normal `LoginModules` just in case the Ldap server is not available or misconfigured, since authentication falls back to the normal one backed by `etc/users.properties` file. Usually only the admin user would be defined in `etc/users.properties`.

Doing the same in **fabric mode** is slightly different and involves modifying the default profile to add the `LoginModule` with a rank greater than 99. A complete example with docker containers for JBoss fuse, ldap server and phpLdapAdmin can be found at:
- https://github.com/valdar/fuseLdapAuthentcation

## JDBC LoginModule
In order to configure JBoss Fuse authentication with jdbc (i.e. authenticating against a database containing USERS and ROLES) follow these steps:
1. Create users and role tables in a supported DB:
```sql
CREATE TABLE USERS (
  USERNAME varchar(255) NOT NULL,
  PASSWORD varchar(255) NOT NULL,
  PRIMARY KEY (USERNAME)
);
CREATE TABLE ROLES (
  USERNAME varchar(255) NOT NULL,
  ROLE varchar(255) NOT NULL,
  PRIMARY KEY (USERNAME,ROLE)
);
```
Here you need one row for each user-role relationship, a possible example of inserting `jdbcfuse` user with password `jdbcfuse` and roles `admin` and `Monitor` would be:
```
INSERT INTO USERS VALUES ("fusejdbc","fusejdbc");
INSERT INTO ROLES VALUES ("fusejdbc","admin");
INSERT INTO ROLES VALUES ("fusejdbc","Monitor");
```
2. Install the JDBC driver for the DB in JBoss Fuse:
```bash
# from karaf console
JBossFuse:karaf@root> install wrap:file:///path/to/jdbcdriver.jar
```
3. Install the datasource exported as an OSGi service:
```bash
# from karaf console
JBossFuse:karaf@root> install blueprint:file://path/to/datasource.xml
```
Example content of `datasource.xml` for MySql DB would be (needs to be adjusted for the specific DB):
```xml
<blueprint xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0">
    <bean class="com.mysql.jdbc.jdbc2.optional.MysqlDataSource" id="mysqlDatasource">
        <property name="serverName" value="db.server.name.or.ip"></property>
        <property name="databaseName" value="shcema.name"></property>
        <property name="port" value="3306"></property>
        <property name="user" value="user"></property>
        <property name="password" value="password"></property>
   </bean>

    <service id="mysqlDS" interface="javax.sql.DataSource" ref="mysqlDatasource">
        <service-properties>
            <entry key="osgi.jndi.service.name" value="jdbc/karafdb"/>
        </service-properties>
    </service>
</blueprint>
```
4. Install the `JdbcLoginModule`:
```bash
# from karaf console
JBossFuse:karaf@root> install blueprint:file://path/to/jdbcjaas.xml
```
Example content of `jdbcjaas.xml` taking in account previous  examples:
```xml
<?xml version="1.0" encoding="UTF-8"?>
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
    <cm:property-placeholder persistent-id="org.apache.karaf.jaas" update-strategy="reload">
        <cm:default-properties>
            <cm:property name="detailed.login.exception" value="false"/>
            <cm:property name="encryption.name" value=""/>
            <cm:property name="encryption.enabled" value="false"/>
            <cm:property name="encryption.prefix" value="{CRYPT}"/>
            <cm:property name="encryption.suffix" value="{CRYPT}"/>
            <cm:property name="encryption.algorithm" value="MD5"/>
            <cm:property name="encryption.encoding" value="hexadecimal"/>
            <cm:property name="audit.file.enabled" value="true"/>
            <cm:property name="audit.file.file" value="$[karaf.data]/security/audit.log"/>
            <cm:property name="audit.eventadmin.enabled" value="true"/>
            <cm:property name="audit.eventadmin.topic" value="org/apache/karaf/login"/>
        </cm:default-properties>
    </cm:property-placeholder>

    <jaas:config name="karaf" rank="200">
        <jaas:module className="org.apache.karaf.jaas.modules.jdbc.JDBCLoginModule"
                     flags="optional">
            datasource = osgi:javax.sql.DataSource/(osgi.jndi.service.name=jdbc/karafdb)
            query.password = SELECT PASSWORD FROM USERS WHERE USERNAME=?
            query.role = SELECT ROLE FROM ROLES WHERE USERNAME=?
            insert.user = INSERT INTO USERS VALUES(?,?)
            insert.role = INSERT INTO ROLES VALUES(?,?)
            delete.user = DELETE FROM USERS WHERE USERNAME=?
            delete.role = DELETE FROM ROLES WHERE USERNAME=? AND ROLE=?
            delete.roles = DELETE FROM ROLES WHERE USERNAME=?
	    detailed.login.exception = true
      </jaas:module>
	        <jaas:module className="org.apache.karaf.jaas.modules.properties.PropertiesLoginModule" flags="optional">
            users = $[karaf.etc]/users.properties
            detailed.login.exception = ${detailed.login.exception}
            encryption.name = ${encryption.name}
            encryption.enabled = ${encryption.enabled}
            encryption.prefix = ${encryption.prefix}
            encryption.suffix = ${encryption.suffix}
            encryption.algorithm = ${encryption.algorithm}
            encryption.encoding = ${encryption.encoding}
        </jaas:module>
        <jaas:module className="org.apache.karaf.jaas.modules.publickey.PublickeyLoginModule" flags="optional">
            users = $[karaf.etc]/keys.properties
            detailed.login.exception = ${detailed.login.exception}
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

    <!-- The Backing Engine Factory Service for the JDBCLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.jdbc.JDBCBackingEngineFactory"/>
    </service>

    <!-- AutoEncryption support -->
    <bean class="org.apache.karaf.jaas.modules.properties.AutoEncryptionSupport"
            init-method="init" destroy-method="destroy">
        <argument>
            <map>
                <entry key="org.osgi.framework.BundleContext" value-ref="blueprintBundleContext"/>
                <entry key="users" value="$[karaf.etc]/users.properties"/>
                <entry key="encryption.name" value="${encryption.name}"/>
                <entry key="encryption.enabled" value="${encryption.enabled}"/>
                <entry key="encryption.prefix" value="${encryption.prefix}"/>
                <entry key="encryption.suffix" value="${encryption.suffix}"/>
                <entry key="encryption.algorithm" value="${encryption.algorithm}"/>
                <entry key="encryption.encoding" value="${encryption.encoding}"/>
            </map>
        </argument>
    </bean>

    <!-- The Backing Engine Factory Service for the PropertiesLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.properties.PropertiesBackingEngineFactory"/>
    </service>

    <!-- The Backing Engine Factory Service for the PublickeyLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.publickey.PublickeyBackingEngineFactory"/>
    </service>

    <service interface="org.apache.karaf.jaas.modules.EncryptionService" ranking="-1">
        <service-properties>
            <entry key="name" value="basic"/>
        </service-properties>
        <bean class="org.apache.karaf.jaas.modules.encryption.BasicEncryptionService"/>
    </service>

</blueprint>
```
There are also the normal `LoginModules` just in case the DB is not available or misconfigured, since authentication falls back to the normal one backed by `etc/users.properties` file. Usually only the admin user would be defined in `etc/users.properties`.

Doing the same in **fabric mode** is slightly different and involves modifying the default profile to add the `LoginModule` with a rank greater than 99. A complete example with docker containers for JBoss fuse, MySql server and phpMyAdmin can be found at:
- https://github.com/valdar/fuseJdbcAuthentcation

## How to recover inaccessible systems
If for some reason playing with `LoginModule` configurations (or for some other problems) the access to the system is lost one can try to regain it by placing the default `LoginModule` configuration in the `FUSE_HOME/deploy` directory with a very high rank to be sure to override all the others. The content of this xml file would do it:
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
    <cm:property-placeholder persistent-id="org.apache.karaf.jaas" update-strategy="reload">
        <cm:default-properties>
            <cm:property name="detailed.login.exception" value="false"/>
            <cm:property name="encryption.name" value=""/>
            <cm:property name="encryption.enabled" value="false"/>
            <cm:property name="encryption.prefix" value="{CRYPT}"/>
            <cm:property name="encryption.suffix" value="{CRYPT}"/>
            <cm:property name="encryption.algorithm" value="MD5"/>
            <cm:property name="encryption.encoding" value="hexadecimal"/>
            <cm:property name="audit.file.enabled" value="true"/>
            <cm:property name="audit.file.file" value="$[karaf.data]/security/audit.log"/>
            <cm:property name="audit.eventadmin.enabled" value="true"/>
            <cm:property name="audit.eventadmin.topic" value="org/apache/karaf/login"/>
        </cm:default-properties>
    </cm:property-placeholder>

    <jaas:config name="karaf" rank="300">
        <jaas:module className="org.apache.karaf.jaas.modules.properties.PropertiesLoginModule" flags="optional">
            users = $[karaf.etc]/users.properties
            detailed.login.exception = ${detailed.login.exception}
            encryption.name = ${encryption.name}
            encryption.enabled = ${encryption.enabled}
            encryption.prefix = ${encryption.prefix}
            encryption.suffix = ${encryption.suffix}
            encryption.algorithm = ${encryption.algorithm}
            encryption.encoding = ${encryption.encoding}
        </jaas:module>
        <jaas:module className="org.apache.karaf.jaas.modules.publickey.PublickeyLoginModule" flags="optional">
            users = $[karaf.etc]/keys.properties
            detailed.login.exception = ${detailed.login.exception}
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

    <!-- AutoEncryption support -->
    <bean class="org.apache.karaf.jaas.modules.properties.AutoEncryptionSupport"
            init-method="init" destroy-method="destroy">
        <argument>
            <map>
                <entry key="org.osgi.framework.BundleContext" value-ref="blueprintBundleContext"/>
                <entry key="users" value="$[karaf.etc]/users.properties"/>
                <entry key="encryption.name" value="${encryption.name}"/>
                <entry key="encryption.enabled" value="${encryption.enabled}"/>
                <entry key="encryption.prefix" value="${encryption.prefix}"/>
                <entry key="encryption.suffix" value="${encryption.suffix}"/>
                <entry key="encryption.algorithm" value="${encryption.algorithm}"/>
                <entry key="encryption.encoding" value="${encryption.encoding}"/>
            </map>
        </argument>
    </bean>

    <!-- The Backing Engine Factory Service for the PropertiesLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.properties.PropertiesBackingEngineFactory"/>
    </service>

    <!-- The Backing Engine Factory Service for the PublickeyLoginModule -->
    <service interface="org.apache.karaf.jaas.modules.BackingEngineFactory">
        <bean class="org.apache.karaf.jaas.modules.publickey.PublickeyBackingEngineFactory"/>
    </service>

    <service interface="org.apache.karaf.jaas.modules.EncryptionService" ranking="-1">
        <service-properties>
            <entry key="name" value="basic"/>
        </service-properties>
        <bean class="org.apache.karaf.jaas.modules.encryption.BasicEncryptionService"/>
    </service>

</blueprint>
```

If for the hot deploy directory has been disabled (you should usually do it in production) then the only option is to search in `FUSE_HOME/data/cache` directory for the faulty `LoginModule` bundle and shutdown the standalone JBoss Fuse instance, delate the bundle then restart JBoss Fuse. **Be aware that if something worng is deleted you may end up in an unusable system.**
