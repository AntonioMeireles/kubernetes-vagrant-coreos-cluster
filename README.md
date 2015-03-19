# kubernetes-vagrant-coreos-cluster
Turnkey **[Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes)**
cluster setup with **[Vagrant](https://www.vagrantup.com)** (1.7.2+) and
**[CoreOS](https://coreos.com)**.

# Current status
> **k8s version 0.13.2 was released early March/19 and doesn't *appear* to be
> behaving here reliably yet**. We're working on it! Right now the highest
> version supposed to run without issues is 0.12.1.
> (see [here](#customization) for notes regarding how to run specific
> k8s versions)

####If you're lazy, or in a hurry, jump to the [TL;DR](#tldr) section.

## Pre-requisites

 * A **MacOS X** or **Linux host**
  * **Microsoft Windows is *not* supported** at the moment.
    Pull Requests to *fix* it are welcome :smile:
 * **[Vagrant](https://www.vagrantup.com)**
 * a supported Vagrant hypervisor
	* **[Virtualbox](https://www.virtualbox.org)** (the default)
	* **[Parallels Desktop](http://www.parallels.com/eu/products/desktop/)**
	* **[VMware Fusion](http://www.vmware.com/products/fusion)** or **[VMware Workstation](http://www.vmware.com/products/workstation)**
 * some needed userland
 	* **kubectl** (required to manage your kubernetes cluster)
 	* **fleetctl** (optional for *debugging* **[fleet](http://github.com/coreos/fleet)**)
 	* **etcdctl** (optional for *debugging* **[etcd](http://github.com/coreos/fleet)**)

### fleetctl, etcdctl, kubectl installation notes

- On **MacOS X** (and assuming you have [homebrew](http://brew.sh) already installed) run...

   `brew install wget fleetctl etcdctl`

- Download the *kubectl* binary into */usr/local/bin*, which should be (and most
probably is) set in your shell's *$PATH*...

   `./kubLocalSetup install`

   Every time you manually set the `KUBERNETES_VERSION` environment variable
   (see [here](#customization) for details), you need to rerun this command so
   that kubernetes related (on host) *userland* matches what is inside the
   cluster.

- Set all needed environment variables in current shell...

   `$(./kubLocalSetup shellinit)`

some points to keep note too:

- If you want to make that persistent across shells and reboots do instead...

   `./kubLocalSetup shellinit >> ~/.bash_profile`
- If you want to validate the environment variables we just set, run...

   `./kubLocalSetup shellinit`
- If you want to persist these changes to ```$PATH```, run...

    `$(./kubLocalSetup shellinit) >> ~/.bashrc`

## Notes about hypervisors

### Virtualbox

**VirtualBox** is the default hypervisor, and you'll probably need to disable its DHCP server
```
VBoxManage dhcpserver remove --netname HostInterfaceNetworking-vboxnet0
```

### Parallels

If you are using **Parallels Desktop**, you need to install **[vagrant-parallels](http://parallels.github.io/vagrant-parallels/docs/)** provider 
```
vagrant plugin install vagrant-parallels
```
Then just add ```--provider parallels``` to the ```vagrant up``` invocations above.

### VMware
If you are using one of the **VMware** hypervisors you must **[buy](http://www.vagrantup.com/vmware)** the matching  provider and, depending on your case, just add either ```--provider vmware-fusion``` or ```--provider vmware-workstation``` to the ```vagrant up``` invocations above.

## Private Repositories

See **DOCKERCFG** bellow.

## Customization
### Environment variables
Most aspects of your cluster setup can be customized with environment variables. Right now the available ones are:

 - **NUM_INSTANCES** sets the number of nodes (minions).

   Defaults to **2**.
 - **UPDATE_CHANNEL** sets the default CoreOS channel to be used in the VMs.

   Defaults to **alpha**.

   ###### While by convenience, we allow an user to optionally consume CoreOS' *beta* or *stable* channels please do note that as both Kubernetes and CoreOS are quickly evolving platforms we only expect our setup to behave reliably on top of CoreOS' _alpha_ channel.
   So, **before submitting a bug**, either in [this](https://github.com/pires/kubernetes-vagrant-coreos-cluster/issues) project, or in ([Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes/issues) or [CoreOS](https://github.com/coreos/bugs/issues)) **make sure it** (also) **happens in the** (default) **_alpha_ channel** :smile:
 - **COREOS_VERSION** will set the specific CoreOS release (from the given channel) to be used.

   Default is to use whatever is the **latest** one from the given channel.
 - **SERIAL_LOGGING** if set to *true* will allow logging from the VMs' serial console.

   Defaults to **false**. Only use this if you *really* know what you are doing.
 - **MASTER_MEM** sets the master's VM memory.

   Defaults to **512** (in MB)
 - **MASTER_CPUS** sets the number of vCPUs to be used by the master's VM.

   Defaults to **1**.
 - **NODE_MEM** sets the worker nodes' (aka minions in Kubernetes lingo) VM memory.

   Defaults to **1024** (in MB)
 - **NODE_CPUS** sets the number of vCPUs to be used by the minions's VMs.

   Defaults to **1**.
 - **BASE_IP_ADDR** sets the lower bound IP addressing block of the VMs to be
   created.

   Default value is **172.17.8**. This turns very handy if for some reason
   one wants to run test clusters in different hypervisors at same time.

 - **DOCKERCFG** sets the location of your private docker repositories (and keys) configuration.

   Defaults to "**~/.dockercfg**".

   You can create/update a *~/.dockercfg* file at any time
   by running `docker login <registry>.<domain>`. All nodes will get it automatically,
   at 'vagrant up', given any modification or update to that file.

 - **ETCD_CLUSTER_SIZE** sets the number of nodes in the default built-in etcd
   cluster.

   Defaults to **3** (or the total number of nodes of the cluster if lower).
 - **KUBERNETES_ALLOW_PRERELEASES** if *true* then the *latest*
   **KUBERNETES_VERSION** is the *latest* available 'pre-release', otherwise is the
   *latest* available 'release'.

   Defaults to *false*. This is IMHO a bit saner than having to constantly
   hard-code specific versions and in any case the *latest* formal kubernetes
   release should be a good and reasonable, reference point.
 - **KUBERNETES_VERSION** defines the specific kubernetes version being used.

   Defaults to *latest* released version.


So, in order to start, say, a Kubernetes cluster with 3 minion nodes, 2GB of RAM and 2 vCPUs per node one just would do...

```
NODE_MEM=2048 NODE_CPUS=2 NUM_INSTANCES=3 vagrant up
```

Please do note that if you were using non default settings to startup your
cluster you *must* also use those exact settings when invoking
`vagrant ssh` to communicate with any of the nodes in the cluster as otherwise
things may not behave as you'd expect.

### Synced Folders
You can automatically mount in your *guest* VMs, at startup, an arbitrary
number of local folders in your host machine by populating accordingly the
`synced_folders.yaml` file in your `Vagrantfile` directory. For each folder
you which to mount the allowed syntax is...

```yaml
# the 'id' of this mount point. needs to be unique.
- name: foobar
# the host source directory to share with the guest(s).
  source: /foo
# the path to mount ${source} above on guest(s)
  destination: /bar
# the mount type. only NFS makes sense as, presently, we are not shipping
# hypervisor specific guest tools. defaults to `true`.
  nfs: true
# additional options to pass to the mount command on the guest(s)
# if not set the Vagrant NFS defaults will be used.
  mount_options: 'nolock,vers=3,udp,noatime'
# if the mount is enabled or disabled by default. default is `true`.
  disabled: false
```

## TL;DR

### Install kubectl

```
./kubLocalSetup install
$(./kubLocalSetup shellinit)
```

### Set-up cluster

```
NODE_MEM=2048 NODE_CPUS=1 NUM_INSTANCES=2 vagrant up
```

This will start the `master` and 2 `minion` nodes. On them a local etcd
cluster will be bootstrapped, Kubernetes binaries will be downloaded and
all needed services started, as a *bonus* a docker mirror cache will be
provisioned in the `master`, to speed up container provisioning. This
can take a little bit depending on your Internet connection speed.

Please do note that, at any time, you can increase the number of running
`minion` VMs by increasing the `NUM_INSTANCES` value in subsequent
`vagrant up` invocations.

You can use `fleet` at any time to check the status of your CoreOS cluster...  
`fleetctl list-machines` should get something along
```
MACHINE     IP            METADATA
dd0ee115... 172.17.8.101  role=master
74a8dc8c... 172.17.8.102  role=minion
c93da9ff... 172.17.8.103  role=minion
```

### Usage

Congratulations! You're now ready to use your Kubernetes cluster.

If you just want to test something simple, start with [Kubernetes examples]
(https://github.com/GoogleCloudPlatform/kubernetes/blob/master/examples/).

For a more elaborate scenario [here]
(https://github.com/pires/kubernetes-elasticsearch-cluster) you'll find all
you need to get a scalable Elasticsearch cluster on top of Kubernetes in no
time.

## Licensing

This work is [open source](http://opensource.org/osd), and is licensed under the [Apache License, Version 2.0](http://opensource.org/licenses/Apache-2.0).
