##### Use MQ gateway for SSL termination

- `gateway-http` does not support support SSL. this means that you can't expose SSL endpoints directly. When dealing with web servers, what you normally do is to put a reverse proxy in front of your plain http endpoints, and configure the reverse proxy to support https, so that the communication towards your systems uses https, and then, once it hits your infrastructure first system (the reverse proxy), it continues in plain unencrypted http. This is typically done with Squid, nginx or similarly dedicated systems, and actually you could couple them with JBoss Fuse.  
  The is though, an alternative in JBoss Fuse Gateway, and it's to use `gateway-mq` profile. Despite the misleading name, `gateway-mq` profile includes a `Detecting` protocol capability, used to support all the advanced transport protocols that you can use in `ActiveMQ`. One of the things `DetectingGateway` does is exactly the one described above: it exposes an HTTPS generic entrypoint that you can directly all your communications to, and then it takes care to forward the communication to a plain HTTP endpoint and to forward the answer, to the corresponding caller, on the HTTPS channel.

example:

```
# generate SSL self-signed certificates

HOSTNAME="localhost" # this clearly depends on your host and the resolver you are using

$JAVA_HOME/bin/keytool -genkeypair \
    -dname "CN=$HOSTNAME, O=name of your organization, L=city name, ST=your state or province, C=US" \
    -validity 365 \
    -alias cert_alias \
    -keypass password \
    -storepass password \
    -keystore paolo.jks \
    -keyalg RSA

 $JAVA_HOME/bin/keytool -list -v \
  -keystore paolo.jks \
  -storepass password

 $JAVA_HOME/bin/keytool -export \
  -alias cert_alias \
  -file server.pub.cert \
  -keystore paolo.jks \
  -storepass password

 $JAVA_HOME/bin/keytool -import \
  -alias cert-alias \
  -file server.pub.cert \
  -keystore trust_store.ts \
  -storepass password
```  

```

# in Fabric mode
fabric:create --wait-for-provisioning --resolver localip
# we are removing jboss-fuse-full since it brings up a broker on port 61616, that is the default one for the detecting gateway
container-remove-profile root jboss-fuse-full 

wait-for-provisioning -v
# add a profile that allows us to deploy a sample cxf http service
container-add-profile root gateway-http  feature-cxf fabric

# create a profile with a sample cxf rest service
# don't mind about the version, 621117 is published on public maven repos, newer versions are not
wait-for-provisioning -v
profile-create test
profile-edit --bundle mvn:io.fabric8.quickstarts/cxf-rest/1.2.0.redhat-621117 test 
container-add-profile root test 
wait-for-provisioning -v

# configure gateway-mq for ssl
profile-edit --pid io.fabric8.gateway.detecting/keyPassword=password gateway-mq
profile-edit --pid io.fabric8.gateway.detecting/keyStorePassword=password gateway-mq
profile-edit --pid io.fabric8.gateway.detecting/keyStoreURL=file:///home/fuse/paolo.jks gateway-mq
profile-edit --pid io.fabric8.gateway.detecting/sslEnabled=true gateway-mq
profile-edit --pid io.fabric8.gateway.detecting/trustStorePassword=password gateway-mq
profile-edit --pid io.fabric8.gateway.detecting/trustStoreURL=file:///home/fuse/trust_store.ts gateway-mq

# assign profile
container-add-profile root  gateway-mq
```

Now you can test your configuration. Your endpoint will be available at 3 different locations:
1. its own direct address
2. http gateway address
3. https (detecting) gateway address

```
# direct endpoint
curl http://localhost:8181/cxf/crm/customerservice/customers/123
# unencrypted gateway
curl http://localhost:9000/cxf/crm/customerservice/customers/123
# encrypted gateway
curl -k https://localhost:61616/cxf/crm/customerservice/customers/123
```

To use the real certificate in curl for the https invocation:
```
openssl s_client -showcerts -connect localhost:61616 |  sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p'  > cert.pem

curl --cacert cert.pem https://localhost:61616/cxf/crm/customerservice/customers/123
```