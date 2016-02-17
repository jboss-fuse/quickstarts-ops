##### official Red Hat documentation
[https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html-single/Security_Guide/index.html#FESBLDAPTutorial]

##### Full demo docker based
[https://github.com/valdar/fuseLdapAuthentcation] by Andrea Tarocchi

This demo provides:

- a OpenLDAP docker image preconfigured with a couple of users
- provides the correct Fuse LoginModule configuration, to match with the structure of the specific OpenLDAP tree
- offers also a preconfigured instance of an LDAP web admin console, to easily view and edit LDAP content.


##### Briefly:

- you need to deploy in Fuse the definition of a `LoginModule` referencing LDAP and its configuration
- you also need to keep a reference to a `ZookeeperLoginModule` that is used internally for internal administrative users
- you need to give to this new `<jaas:config>` element a rank higher than the default one that comes with Fuse, using `100` works.
- you need to adapt the configuration to your specific LDAP structure. (ex. `user.base.dn=ou=People,dc=example,dc=org` might be different in your organization.)
- you can use this log level to have more information about LDAP interaction: `org.ops4j.pax.logging/log4j.logger.org.apache.karaf.jaas=DEBUG`
- if you need further examples you can compare the following 2 configurations:
   - https://github.com/valdar/fuseLdapAuthentcation/blob/master/ldap-module.xml
   - https://github.com/apache/karaf/blob/master/jaas/modules/src/test/resources/org/apache/karaf/jaas/modules/ldap/ldap.properties

##### Steps

1) define a new xml file containing `LoginModule` definition (note support for `${ldaphostserver}` as an externalized property.):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:jaas="http://karaf.apache.org/xmlns/jaas/v1.0.0"
  xmlns:ext="http://aries.apache.org/blueprint/xmlns/blueprint-ext/v1.0.0"
  xmlns:cm="http://aries.apache.org/blueprint/xmlns/blueprint-cm/v1.1.0"
           xsi:schemaLocation="http://www.osgi.org/xmlns/blueprint/v1.0.0 http://www.osgi.org/xmlns/blueprint/v1.0.0/blueprint.xsd
				http://aries.apache.org/blueprint/xmlns/blueprint-cm/v1.1.0 http://aries.apache.org/schemas/blueprint-cm/blueprint-cm-1.1.0.xsd
				http://camel.apache.org/schema/blueprint http://camel.apache.org/schema/blueprint/camel-blueprint.xsd">


  <cm:property-placeholder persistent-id="ldap.server" update-strategy="none" >
    <cm:default-properties>
      <cm:property name="ldaphostserver" value="localhost"/>
    </cm:default-properties>
  </cm:property-placeholder>
  <jaas:config name="karaf" rank="100">
    <jaas:module className="io.fabric8.jaas.ZookeeperLoginModule" flags="sufficient"/>
    <jaas:module className="org.apache.karaf.jaas.modules.ldap.LDAPLoginModule" flags="sufficient">
      initialContextFactory=com.sun.jndi.ldap.LdapCtxFactory
      connection.username=cn=admin,dc=example,dc=org
      connection.password=admin
      connection.protocol=
      connection.url=ldap://${ldaphostserver}:389
      user.base.dn=ou=People,dc=example,dc=org
      user.filter=(uid=%u)
      user.search.subtree=true
      role.base.dn=ou=Groups,dc=example,dc=org
      role.name.attribute=cn
      role.filter=(memberuid=%u)
      role.search.subtree=true
      authentication=simple
    </jaas:module>
  </jaas:config>
</blueprint>
```
2) add the externalized configuration:
```bash
profile-edit --pid ldap.server/ldaphostserver=172.17.0.2   default
```
3) add the the defined module to the `default` profile (the example assumes the file was inside `jboss-fuse-6.2.*/fabric/import/fabric/profiles/default.profile` at fabric creation time, but you can use any kind of supported artifact coordinates to point to the file. )
```bash
profile-edit --bundle blueprint:profile:ldap-module.xml default
```
