Official documentation: [https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html/Fabric_Guide/Git-External.html]

JBoss Fuse uses an internal git repository to store information about two important areas:
* Fabric runtime configuration - fabric versions, profiles associated with the versions, etc.
* Patch mechanism - history of patching process used to correctly determine what files should be included in patched version, rollback handling, etc.

By default, the Fabric git repository is stored in _<fuse_home>/data/git/local/fabric_ directory, but you can override this location with the
fabric:create command to point to any other external repository.

Fabric:create command uses three different properties to specify the external repository:
* *--external-git-url* - defines the URL to the external git repository,
* *--external-git-user* - defines the external git username (in case of credentials based authentication). In the GitHub example this will be your
GitHub username,
* *--external-git-password* - defines the external git password (in case of credentials based authentication). In the GitHub example this will be your
GitHub password.

There are currently (6.2.1 version of JBoss Fuse) two options of authentication to git:
* Using SSH public key
* Using credentials

**NOTE**: Unfortunately, JBoss Fuse currently does not support password encrypted SSH keys.

### Configuring environment

If you want to use SSH key authentication, add your _public_ SSH key (its default location is _~/.ssh/id_rsa.pub_) to your (GitHub profile)
[https://github.com/settings/ssh]. Then you should test if your connection is working using the ssh command. You should see an output similar to
this one:

```bash
myuser@myhost:~$ ssh git@github.com
Hi myuser! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

Then create a new GitHub repository and keep your eye on the connection URLs, because we will need that later. Let's say our connection strings are:
* HTTPS: https://github.com/gituser/fabric.git
* SSH: git@github.com:gituser/fabric.git

### Configuring JBoss Fuse

We start with the fresh copy of JBoss Fuse 6.2.1 - we unzip the file *jboss-fuse-full-6.2.1.redhat-084.zip* to desired location and uncomment the
default admin user (remove the # sign in <fuse_directory>/etc/users.properties). Then we can start JBoss Fuse in interactive mode using
<fuse_directory>/bin/fuse.

As you already know, there are two authentication options. Based on the method you choose, use one of the following commands:

* For credentials based authentication
```bash
JBossFuse:karaf@root> fabric:create --external-git-url https://github.com/gituser/fabric.git --external-git-user gituser --external-git-password
gitpassword --wait-for-provisioning
```

* For SSH key based authentication
```bash
JBossFuse:karaf@root> fabric:create --external-git-url git@github.com:gituser/fabric.git --wait-for-provisioning
```

After the container is provisioned successfully, you can check your repository in GitHub. You should already see a new branch called *1.0* which
contains all the profiles defined in the 1.0 version of fabric.

For more info about the Git in Fabric, please see the (official documentation)[https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2
.1/html/Fabric_Guide/Git.html#Git-HowItWorks] about this topic.