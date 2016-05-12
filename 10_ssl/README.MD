JBoss Fuse consists of many components which support SSL.
You might want for instance use different SSL certificate for your WebConsole than for traffic between MQ brokers.
To allow this fine grained control, Fuse provides several places to configure SSL.

### Working with Java keytool <a id="java_keytool">&nbsp;</a>

> Introduction to X.509 certificate management can be found in documentation chapter [Managing Certificates](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html/Security_Guide/ManageCerts.html).
It describes steps to create root CA with OpenSSL, generation and signature of certificates and their import back to Java keystore.

For testing purposes Java keystore with self-signed certificate can be generated with following command:
> If you omit `-dname` you will be asked to fill informations interactively.  
If you omit `-dkeypass` you will be asked to confirm key password same as store password.

```bash
# create new keystore (or add keypair with unique alias)
$JAVA_HOME/bin/keytool -genkeypair \
    -dname "CN=name of your container, O=name of your organization, L=city name, ST=your state or province, C=US" \
    -validity 365 \
    -alias cert-alias \
    -keypass awesomeKeyPass \
    -storepass awesomeStorePass \
    -keystore broker.ks \
    -keyalg RSA
```
Show contents of keystore
```bash
# and optionally display content of keystore
$JAVA_HOME/bin/keytool -list -v \
  -keystore broker.ks \
  -storepass awesomeStorePass
```

Export public certificate (client needs to import it to it's trust store)
```bash
$JAVA_HOME/bin/keytool -export \
  -alias cert-alias \
  -file server-pub-cert \
  -keystore broker.ks \
  -storepass awesomeStorePass
```

Create trust store which will trust server's certificate:
```bash
$JAVA_HOME/bin/keytool -import \
  -alias cert-alias \
  -file server-pub-cert \
  -keystore broker.ts \
  -storepass awesomeStorePass
```


### Server side configuration
This is a brief list of SSL configurable endpoints exposed by Fuse:
* Jetty / Hawtio WebConsole ([official documentation](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html/Security_Guide/WebConsole.html))
  * default http://localhost:8181/
  * make it https://localhost:${pick_port}
* Transports of JBoss A-MQ running in Fuse ([official documentation](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_A-MQ/6.2/html-single/Managing_and_Monitoring_a_Broker/index.html#AMQSSL))
  * default transport `openwire`
  * make it `ssl` transport


#### Jetty / Hawtio WebConsole
**Configured in `$FUSE_HOME/etc/jetty.xml`**

TODO - there changes between fuse 6.2 and 6.3

#### Transports of JBoss A-MQ running in Fuse
**Configured in `$FUSE_HOME/etc/activemq.xml`**

Create keystore and truststore in `$FUSE_HOME/conf`  _(update sslContext paths if you decide to store keystore in different location)_.

Add keystore configuration to `activemq.xml`.
```xml
<beans>
  <broker>
    ...
    <!-- add sslContext element to broker section -->
    <sslContext>
    	<sslContext keyStore="file:${karaf.home}/conf/broker.ks"
    				keyStorePassword="awesomeStorePass"
    				keyStoreKeyPassword="awesomeKeyPass"
    				trustStore="file:${karaf.home}/conf/broker.ts"
    				trustStorePassword="awesomeStorePass"
    			/>
    </sslContext>
     ...
  </broker>
</beans>
```

> Notes
* Default value for `keyStoreKeyPassword` is equal to `keyStorePassword`.
* Put only one key to keystore.
* You can put multiple keys to truststore.

Switch openwire transport to ssl.
```xml
<!-- find existing transportConnectors section -->
<transportConnectors>
  <!-- default -- delete/comment out openwire to disable unencrypted traffic -->
  <transportConnector name="openwire" uri="tcp://${bindAddress}:${bindPort}"/>

  <!-- add encrypted transport, you may use ${bindPort} placeholder -->
  <transportConnector name="ssl" uri="ssl://${bindAddress}:61617?needClientAuth=true"/>
</transportConnectors>
```
> Properties bindAddress and bindPort come from `$FUSE_HOME/etc/system.properties` `activemq.host` and `activemq.port` respectively.

Restart Fuse to pickup changes in `activemq.xml`.

You verify SSL certificate of transport connector by running:
```bash
openssl s_client -showcerts -connect localhost:61617 </dev/null

# it will print lot of things, among others your certificate and it's issuer
# (this one is self-signed so they're equal):
Certificate chain
 0 s:/C=US/ST=your state or province/L=city name/O=name of organization/CN=name of container
   i:/C=US/ST=your state or province/L=city name/O=name of organization/CN=name of container
```

#### SSL for JMX

- official docs: https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html/Security_Guide/FMQSecurityJmxSSL.html

##### Test
```bash
cd $FUSE_HOME/etc
jconsole -J-Djavax.net.debug=ssl -J-Djavax.net.ssl.trustStore=jbossweb.keystore -J-Djavax.net.ssl.trustStoreType=JKS -J-Djavax.net.ssl.trustStorePassword=JbossPassword
# pass this config: service:jmx:rmi://localhost:44444/jndi/rmi://localhost:1099/karaf-root  admin/ admin
```

##### Standalone
```bash
FUSE_HOME=$(pwd)
cd $FUSE_HOME/etc
keytool -genkey -v -alias jbossalias -keyalg RSA -keysize 1024 -keystore jbossweb.keystore -validity 3650 -keypass JbossPassword -storepass JbossPassword -dname "CN=127.0.0.1, OU=RedHat Software Unit, O=RedHat, L=Boston, S=Mass, C=USA"

cat > $FUSE_HOME/etc/keystore.xml <<EOF
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
           xmlns:jaas="http://karaf.apache.org/xmlns/jaas/v1.0.0">
    <jaas:keystore name="sample_keystore"
                    rank="1"
                    path="file:etc/jbossweb.keystore"
                    keystorePassword="JbossPassword"
                    keyPasswords="jbossalias=JbossPassword" />
</blueprint>
EOF

cp $FUSE_HOME/etc/keystore.xml $FUSE_HOME/deploy/


cat >> $FUSE_HOME/etc/org.apache.karaf.management.cfg <<EOF
secured = true
secureProtocol = TLSv1
keyAlias = jbossalias
keyStore = sample_keystore
trustStore = sample_keystore
keyStoreAvailabilityTimeout = 30000
EOF

# eventually change the startup level for the keystore
osgi:bundle-level keystore.xml 29
```

##### Fabric

```bash
# in bash

FUSE_HOME=$(pwd)
cd $FUSE_HOME/etc
keytool -genkey -v -alias jbossalias -keyalg RSA -keysize 1024 -keystore jbossweb.keystore -validity 3650 -keypass JbossPassword -storepass JbossPassword -dname "CN=127.0.0.1, OU=RedHat Software Unit, O=RedHat, L=Boston, S=Mass, C=USA"

cat > $FUSE_HOME/etc/keystore.xml <<EOF
<blueprint xmlns="http://www.osgi.org/xmlns/blueprint/v1.0.0"
           xmlns:jaas="http://karaf.apache.org/xmlns/jaas/v1.0.0">
    <jaas:keystore name="sample_keystore"
                    rank="1"
                    path="file:etc/jbossweb.keystore"
                    keystorePassword="JbossPassword"
                    keyPasswords="jbossalias=JbossPassword" />
</blueprint>
EOF

cd $FUSE_HOME
bin/fuse



# in karaf shell

fabric:create --wait-for-provisioning

profile-create auth_test
profile-edit --bundle 'blueprint:file://${karaf.etc}/keystore.xml;start-level=29' auth_test
profile-edit --pid org.apache.karaf.management/secured=true auth_test
profile-edit --pid org.apache.karaf.management/secureProtocol=TLSv1 auth_test
profile-edit --pid org.apache.karaf.management/keyAlias=jbossalias auth_test
profile-edit --pid org.apache.karaf.management/keyStore=sample_keystore auth_test
profile-edit --pid org.apache.karaf.management/trustStore=sample_keystore auth_test
profile-edit --pid org.apache.karaf.management/keyStoreAvailabilityTimeout=30000 auth_test

container-add-profile root auth_test
```
