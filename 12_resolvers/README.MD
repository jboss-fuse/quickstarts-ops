[Official documentation about resolvers](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html/Console_Reference/ConsoleFabricContainerResolverSet.html)

##### Description
An operating system instance can have multiple ip addresses. For this reason, the answer to "what is that box ip address?" is not that trivial.  
On top of that you can address a box by hostname.

For all of these reasons, Fabric allows to configure the `resolver` property for each container instance.

Think at the `resolver` property as **"the way, that instance, likes to be called"**.

The list of resolvers available are:

- `localip`, `localhostname`, `publicip`, `publichostname`,  `manualip`


The important things to notice are:

- default policy is `localhostname` and with that you "lose some control", since if you have more than one ip assigned, it's again not that straight forward to guess which of the various ip you will resolve to (more in general hostname-ip is actually a MANY_TO_MANY relationship, despite people tend to forget that). You should verify what your hostname ip resolves to.

- `manualip` is the only strategy that requires an additional parameter at creation time. The **additional parameter** is the desired ip address. If the ip addres needs to be changed after creation time, a manual edit on ZK registry is required, as specified in `fabric:container-resolver-set --help`


##### Gotchas

There are scenarios where you can lose access to your Fabric due ip and networking related issues.  
The main problem is that the strategy (`resolver` + ip) you define at Fabric creation time, can conflict with the current status of you networking configuration.  

If for example you have a Fabric configured with `manualip` and `10.0.0.1`, that might be the ip you have when connected to your LAN/WIFI and you get disconnected for quite a long time you can lose Fabric functionality. Why is that?  
Because you explicitely said that "your root node likes to be called 10.0.0.1". But if that ip was assigned by DHCP and there is no longer any DHCP, your system most probably will release that ip address, and when your middleware tries to talk with `10.0.0.1` it won't find any matching for it.  

The described issue is quite typical in VMs scenarios, or when you have started a Fabric with default resolver and you went offline or joined another network.

The strategies to control this depend on you requirement, here is described one based on how to solve the issue when working with docker in vm mode:

```bash
##### example session
docker run --name fuse --rm -it fuse:6.2

# assigned ip:
# 172.17.0.6

# start fuse
bin/fuse

# create fabric
fabric:create --wait-for-provisioning --resolver manualip -m 127.0.0.1

JBossFuse:karaf@root> container-resolver-list
[id]  [resolver]  [local hostname]  [local ip]  [public hostname]  [public ip]  [manual ip]
root  manualip    825d3948bbc8      172.17.0.6                                  127.0.0.1  



# create docker base image from runtime
docker commit fuse fabric

# start a new docker container, with a fabric already configured
docker run --rm --name fabric1 -it fabric
# new ip is 172.17.0.7

BossFuse:karaf@root> container-resolver-list
[id]  [resolver]  [local hostname]  [local ip]  [public hostname]  [public ip]  [manual ip]
root  manualip    d19fbbd95b81      172.17.0.7                                  127.0.0.1  

# change resolver strategy to use the new "real" ip address
container-resolver-set localip
# or keep manual ip, but to change the ip you have to write directly on zk. See instructions in `container-resolver-set --help`

JBossFuse:karaf@root> container-resolver-list
[id]  [resolver]  [local hostname]  [local ip]  [public hostname]  [public ip]  [manual ip]
root  localip     d19fbbd95b81      172.17.0.7                                  127.0.0.1
``
