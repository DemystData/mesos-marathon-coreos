# Running Marathon, Mesos, and Zookeeper on CoreOS in HA

In this guide we're going to lean on [Fleet](https://coreos.com/using-coreos/clustering/) and [Etcd](https://coreos.com/using-coreos/clustering/) to run and coordinate Zookeeper, Mesos Master/Slaves, and Marathon instances running in Docker containers.

We're running these examples on a MacBook Pro, using [CoreOS on Vagrant](https://coreos.com/docs/running-coreos/platforms/vagrant/), CoreOS 633.1.0, Zookeeper 3.4.6, Mesos 0.22.0, and Marathon 0.8.1.

This solution is draws upon the excellent posts by Livewyer (1) on running Marathon and Mesos in Docker on CoreOS, and Nick Randell (2) on running Zookeeper on CoreOS, and countless other documents, emails, source code, and github issue comments.

For help using Marathon once you've completed this guide, see the https://mesosphere.github.io/marathon/docs/application-basics.html

All the listings in this guide are available on the Github:

- https://github.com/cmbaron/docker-zookeeper
- https://github.com/cmbaron/mesos-marathon-coreos

The examples assume you're running a three node CoreOS cluster and have cloned the mesos-marathon-coreos repository onto the first node as follows:

   core@core-01 ~ $ git clone https://github.com/cmbaron/mesos-marathon-coreos

## Zookeeper Ensemble

### Generating the Zookeeper Unit Files

Zookeeper's configuration requires hard-coding the location of members of the Ensemble so it cannot be subjected to the whimsy of fleet's scheduler.  To accommodate this, Nick Randell wrote the configure script which generates fleet unit files exposing the peers in an environment variable, and pegging each instance to a particular machines using the fleet MachineID directive.

We extend Nick's script and template to configure Zookeeper using IP addresses, and add sidekick units to expose Zookeeper's address and port into Etcd.

### Example launching and running on a three node cluster

    core@core-01 ~ $ cd mesos-marathon-coreos/zookeeper
	core@core-01 ~/mesos-marathon-coreos/zookeeper $ ./configure 3
	172.17.8.103,172.17.8.101,172.17.8.102
	Setting server 1 as 0f5fe33feebd42c09cf49c4397e098b5 (172.17.8.103)
	Setting server 2 as 6891d7331d694e6296d4e39354490c7d (172.17.8.101)
	Setting server 3 as 8211f3b082fc40be9373121f05853707 (172.17.8.102)
	
	core@core-01 ~/mesos-marathon-coreos/zookeeper $ fleetctl start *@*
	Unit zookeeper-discovery@2.service launched on 6891d733.../172.17.8.101
	Unit zookeeper@2.service launched on 6891d733.../172.17.8.101
	Unit zookeeper@3.service launched on 8211f3b0.../172.17.8.102
	Unit zookeeper@1.service launched on 0f5fe33f.../172.17.8.103
	Unit zookeeper-discovery@1.service launched on 0f5fe33f.../172.17.8.103
	Unit zookeeper-discovery@3.service launched on 8211f3b0.../172.17.8.102
	
	core@core-01 ~/mesos-marathon-coreos/zookeeper $ fleetctl list-units
	UNIT				MACHINE				ACTIVE	SUB
	zookeeper-discovery@1.service	0f5fe33f.../172.17.8.103	active	running
	zookeeper-discovery@2.service	6891d733.../172.17.8.101	active	running
	zookeeper-discovery@3.service	8211f3b0.../172.17.8.102	active	running
	zookeeper@1.service		0f5fe33f.../172.17.8.103	active	running
	zookeeper@2.service		6891d733.../172.17.8.101	active	running
	zookeeper@3.service		8211f3b0.../172.17.8.102	active	running
	
	core@core-01 ~ $ etcdctl get /services/zookeepers
	172.17.8.103:2181,172.17.8.101:2181,172.17.8.102:2181
	
### Docker Image

The docker image and start script derive from Nick Randell's post.  Here we modify it to use Oracle's Java and add 'quorumListenOnAllIPs=true' to the configuration template.

We add 'quorumListenOnAllIPs=true' in the zoo.cfg.template so the Zookeeper will listen for elections on the bridged IP visible within Docker container.  

## Mesos Master

Here we allow Fleet to schedule a user-specified number of mesos-master instances anywhere in the CoreOS Cluster.  We'll use the mesos-master image on DockerHub provided by Mesosphere.

The Docker config comes from the Livewyer Blog post (1), but modified to eliminate the intermediate Docker container and Consul dependency.

### An example starting three instances:

    core@core-01 ~/mesos-marathon-coreos $ fleet submit mesos-master@.service
    core@core-01 ~/mesos-marathon-coreos $ fleet start mesos-master@1.service mesos-master@2.service mesos-master@3.service
    Unit mesos-master@1.service launched on 0f5fe33f.../172.17.8.103
    Unit mesos-master@2.service launched on 6891d733.../172.17.8.101
    Unit mesos-master@3.service launched on 8211f3b0.../172.17.8.102

## Mesos Slaves

We'll utilize a global unit to start the Mesos slave on all nodes in the cluster.  The Mesos slaves discover the Master cluster through the Zookeeper ensemble discoverable at the Etcd key /services/zookeepers.

### Starting it up

    core@core-01 ~/mesos-marathon-coreos $ fleet start mesos-slave.service
    Triggered global unit mesos-slave.service start


## Marathon Masters

Marathon discovers Mesos master cluster through the Zookeeper ensemble discoverable at the Etcd key /services/zookeepers.
By setting the hostname to the value of COREOS_PUBLIC_IPV4 the Marathon instances are able to proxy requests to the master since the master will announce itself as the value of 'hostname'.

Starting three instances, and discovery sidekicks to export the marathon locations into etcd:

    core@core-01 ~/mesos-marathon-coreos $ fleet submit marathon@.service
    core@core-01 ~/mesos-marathon-coreos $ fleet start marathon@1.service marathon@2.service marathon@3.service marathon-discovery@1.service marathon-discovery@2.service marathon-discovery@3.service
    Unit marathon@1.service launched on 0f5fe33f.../172.17.8.103
    Unit marathon@2.service launched on 6891d733.../172.17.8.101
    Unit marathon@3.service launched on 8211f3b0.../172.17.8.102
    Unit marathon-discovery@1.service launched on 0f5fe33f.../172.17.8.103
    Unit marathon-discovery@2.service launched on 6891d733.../172.17.8.101
    Unit marathon-discovery@3.service launched on 8211f3b0.../172.17.8.102

We can use bridged networking by setting the LIBPROCESS_PORT environment and mapping that port into the container.

## Accessing Marathon and Mesos UIs

If you're using the VMs created by the coreos-vagrant project, then you should be able to point your web browser at http://172.17.8.101:8080/ to reach the Marathon UI and http://172.17.8.101:5050 to reach the Mesos UI.


