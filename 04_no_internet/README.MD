##### Consideration
The main interactions that JBoss Fuse has with the outer internet are:
- Downloading Maven artifacts that are not available locally
- Hitting public online services. (ex. freegeoip, see [ENTESB-4164](https://issues.jboss.org/browse/ENTESB-4164))
- Opening direct outgoing HTTP connections (see [ENTESB-4489](https://issues.jboss.org/browse/ENTESB-4489))

##### Usual checks
- verify if DNS resolution works and doesn't hang
- verify that outgoing http connections don't hang
- verify that all the Maven artifacts you need are available offline
- verify that your setup is pointing to the correct Maven repositories
- consider to remove public Maven repositories from your configuration (to avoid check that will always fail.)


##### ConfigAdmin entries related to Maven that contain by default services requiring internet access:
- `org.ops4j.pax.url.mvn/org.ops4j.pax.url.mvn.repositories` (this is for Standalone)
- `io.fabric8.maven/io.fabric8.maven.repositories`
- `io.fabric8.agent/org.ops4j.pax.url.mvn.repositories`

##### useful JVM network settings
- **Offline** status is not very well defined. At the end it means that the environment will never be able to access resources outside its LAN, but the way this effect is obtain can be vary. One of the main issues here is the default firewall policy that can be enforced:
   - `REJECT` : the invoker is immediately notified by the firewall of the impossibility to reach the required resources
   - `DROP` : the firewall ignores the invocation (with the result that the client will never reached the requested resource) but it doesn't notify the client of this. So the client, thinks that the resource just requires some time, and patiently wait for it. It wait until the JVM configured timeout will be hit.
 - JVM settings controlling tcp connections behavior are described [here](https://docs.oracle.com/javase/7/docs/technotes/guides/net/properties.html):
    - `sun.net.client.defaultConnectTimeout`
    - `sun.net.client.defaultReadTimeout`

##### Alternative options to speed up things if you don't have direct control on the firewall
- specify JVM settings at boot (ex. `EXTRA_JAVA_OPTS="-sun.net.client.defaultConnectTimeout=2000" bin/fuse debug`)
- edit `/etc/hosts` file to alter DNS resolution for unaccessible remote services (ex. `127.0.0.1    freegeoip.net`)

##### example iptables rules to replicate an offline environment:
```bash
sudo iptables -I FORWARD -j REJECT -s 172.17.0.0/16 ! -d 172.17.0.0/24 --protocol tcp -m comment --comment "Simulate offline env for JBoss Fuse"
```

Notes:

- the policy has been set to `REJECT`. The alternative `DROP` would let the tcp connections to slowly die only after the JVM globally configured timeout
- the protocol is specified to `tcp`. Omitting that param will apply the blocking rule even to DNS resolutions, generating a different kind of reported issues
- you might want to use a chain other than `FORWARD`, depending on your network topology


##### Inspect, externally, all the outgoing traffic.
Tools like `tcpdump` can be used to inspect all the outgoing traffic.  
When you have to monitor just **HTTP** traffic, it might be easier to deploy a transparent proxies that intercepts all the real outgoing http traffic for log (or caching) purposes.

ex.
```bash
# Intercepts only packets from docker, since the prerouting table is not traversed by packets originating on the same host

### run squid passtrough container ###
docker run -d --net host --privileged --name squid -e MAX_CACHE_OBJECT=1024 jpetazzo/squid-in-a-can # port 3128 and 3129 open on host

### add routes ###
sudo iptables -t nat -A PREROUTING -s 172.17.0.0/16 -p tcp --dport 80 -j REDIRECT --to 3129 -w -m comment --comment "Intercept Docker containers outgoing traffic and redirects it to Squid. Squid needs to be running"
sudo iptables -t nat -A PREROUTING -s 172.17.0.0/16 -p tcp --dport 443 -j REDIRECT --to 3129 -w -m comment --comment "Intercept Docker containers outgoing traffic and redirects it to Squid. Squid needs to be running"
sudo iptables -I INPUT -s 172.17.0.0/16 -j ACCEPT -m comment --comment "Accept traffic from containers to host services, needed for Squid"



### check squid logs ###
docker exec -it squid tail -F /var/log/squid3/access.log

### test to be invoked from host, it doesn't work directly from the host, since normal traffic is not pre-routed. ssh traffic is though, so i can invoke my socks, that will be interecepted by the iptables

curl -L --proxy localhost:3128 www.google.com
```
