This project contains CoreOS Unit files needed to run a HA zookeeper, mesos and marathon clusters.

Example launching and running on a three node cluster:

    core@core-01 ~ $ git clone https://github.com/DemystData/mesos-marathon-coreos.git
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

    core@core-01 ~/mesos-marathon-coreos/zookeeper $ etcdctl get /services/zookeepers
    172.17.8.103:2181,172.17.8.101:2181,172.17.8.102:2181

    core@core-01 ~/mesos-marathon-coreos/zookeeper $ cd ..
    core@core-01 ~/mesos-marathon-coreos $ fleetctl submit *.service
    core@core-01 ~/mesos-marathon-coreos $ fleetctl start mesos-master@1.service mesos-master@2.service mesos-master@3.service mesos-slave.service marathon@1.service marathon@2.service marathon@3.service marathon-discovery@1.service marathon-discovery@2.service marathon-discovery@3.service
    Unit mesos-master@1.service launched on 0f5fe33f.../172.17.8.103
    Unit mesos-master@2.service launched on 6891d733.../172.17.8.101
    Unit mesos-master@3.service launched on 8211f3b0.../172.17.8.102
    Unit marathon@1.service launched on 0f5fe33f.../172.17.8.103
    Unit marathon@2.service launched on 6891d733.../172.17.8.101
    Unit marathon@3.service launched on 8211f3b0.../172.17.8.102
    Unit marathon-discovery@1.service launched on 0f5fe33f.../172.17.8.103
    Unit marathon-discovery@2.service launched on 6891d733.../172.17.8.101
    Unit marathon-discovery@3.service launched on 8211f3b0.../172.17.8.102
    Triggered global unit mesos-slave.service start

Now watch fleetctl list-units until all units are up and running.
You may watch the logs using journalctl -f

## Credit

Mesos/Marathon docker configurations loosely based on http://www.livewyer.com/blog/2015/02/16/running-apache-mesos-inside-docker-coreos
Zookeeper cluster extended from https://github.com/nrandell/zookeeper-coreos-cluster licensed under MIT License.  See zookeeper/LICENSE

