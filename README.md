# Home cluster goes BRRRRRRRRRRRRRRRRRR

## Why
Two reasons. First - I run a relatively complex smarthome setup, use some media retrieval software, store data etc. At the end of the day I have about 20 docker containers running from about 10 docker-compose files and managing the stuff became not very funny.
Most important second reason - I wanted to learn k8s at semi-productional level.
## Requirements
1. Manageable and observable workload, current and potential new
2. Fast recovery from hardware issues (re-plug zwave controller and/or drive). 
3. Easy replacement of a machine
4. Automatic ofsite backup of some of my storage
## Hardware
I've decided not to use single-board things like RPis but to use oudated Intel mini-desktops (avaiable for about 100 euro). I already had one and bought two more. RPis have not enough memory, SD cards are killed by intensive IO pretty fast and I wanted to have terabyte-scale storage without having a dedicated NAS.
So ended up with Asus Vivo mediacomputer, Fujitsu mediabox and HP Elite 800. They have similar footprint, all in theory support 2 2.5 inch HDDs but Asus needed a special sled, Fuji required mSATA and HP uses Streamline SATA. So disk setup is a mess, all of them have 2nd HDD installed not in a pretty way. Fuji also had 100Mb Ethernet port so I had to buy a miniPCI Ethernet card. 
Now all machines have 0.8-2T HDD each, 1GbE Ethernet, 8Gb RAM. They are powered through a smart socket which reports that total power usage is between 40-100 Watt.
## Platform
### OS
I use Ubuntu 22.04LTS as a base OS. There are no specific reasons for that, just a well-supported and convenient distro. Machines are installed as 'ubuntu-server'. For disk setup I've created two VGs - vg_root and vg_opt using both SSDs. Root is 100Gi, rest goes to opt. The only exception is Asus which was installed long time ago and has a big media collecton. So Asus has a single VG with 50Gb root and 800Gb opt.
Smarthome has both zwave and zigbee networks so Asus has two USB controllers for those networks.
### Kubernetes
I've decided to go with K3S, mostly to save RAM. I could go with vanilla k8s as well as I understand.
Initially I've set up Master on Fujitsu with mostly default parameters (only `traefik` disabled). However pretty soon I've realized that single-master k3s cluster uses Sqlite DB to store all the cluster data. 

K3s allows easily to switch cluster datastore to an external SQL db or to embedded etcd.

SqliteDB and external DB are single points of failure so I've decided to run etcd and make all nodes masters. In this setup I can remove any node and still have operational cluster, which can be joined by a replacement machine. So I can survive loss of any single node without much problems.

### Cluster setup
#### Network
Cluster uses `flannel` CNI - that's the default and seems to be suited my situation pretty well. `K3s` contains a built-in load balancer service controller (can be replaced by metalLB for example).

One of the pains of the previous setup was need to remember what port (and sometimes what host) stuff listens on. Zwave2MQTT sits on http://rpi2:8999 - not easy to remember. So I delegated one of my domains to `DigitalOcean` and set up `ExtDNS` controller to manage records. Now I should use http://zwave2mqtt.mycluster.net - much better. 
Of course all A records return private addresses of my cluster machines - nginx uses loadbalancer service to listen on all 3 nodes.

Certificate manager was configured to use letsencrypt-issuer. So if `Ingress` contains `tls` data a certificate from LE will be auto-requested and https will be enabled. That was surprisingly easy to set up.

#### Storage
Storage setup was more interesting. Using HostPath nodes was out of question (too boring). Since I've seen `topolvm` before I've decided to use it. Two of cluster nodes had a big LVM VG so I installed and configured topolvm. Part of the installation is modification of cluster scheduler - so that it could take into account remaining capacity of topolvm VG. Took me quite a lot to install this config into k3s, ended up with:
```
ExecStart=/usr/local/bin/k3s \
    server \
    --kube-scheduler-arg=config=/usr/local/etc/k3s/scheduler-config.yaml
```
in k3s.service unit file.

Topolvm worked pretty well but soon I've realized that the requirement of being able to turn off one machine without issues is not satisfied anymore - all the pods using storage were pinned to the nodes were their LVs were created.

For backup I've used `k8up` which went through all PVs having "`backup: true`" label and dumped them to S3 bucket at AWS.

To keep flexibility I've decided to try `Longhorn` CSI. 

Longhorn is a fantastic storage manager. Basically how it works: 
1. All enabled nodes export their storage as iSCSI
2. When a PV is requested:
   1. a new `engine` pod is created on the node where the volume is going to be mounted
   2. it mounts an iSCSI target as a block device somewhere
   3. It mounts this block device so that pod can access it
3. Also it keeps several replicas of data and creates snapshots.

So Longhorn enables me to have pods using drives from other nodes and to have replicas of my data on different nodes. That fully satisfied the requirement of being able to kill one of the nodes.

```
 fio --randrepeat=1 --ioengine=libaio --direct=1 --gtod_reduce=1 --name=test --filename=random_read_write.fio --bs=4k --iodepth=64 --size=4G --readwrite=randrw --rwmixread=75
```

physical SSD: `[r=60.1MiB/s,w=19.8MiB/s][r=15.4k,w=5062 IOPS]`

Longhorn with 2 replicas: `[r=30.3MiB/s,w=10.2MiB/s][r=7767,w=2612 IOPS]`

So Longhorn volumes are exactly twice as slow as physical drives. I think I can leave with that.

Most of Longhorn configuration is done via GUI `values.yaml` is quite big and complicated.

**If you try to uninstall longhorn be sure uninstall is enabled in settings.** Otherwise you'll end up with a lot of objects in "Terminating" state waiting for a finalizers
which are already gone.  

Longhorn has built-in backup system using S3 as a backend. Since it works at block level it can do actual incremental backups. 

At the end I have two longhorn storage classes - with 1 and 2 replicas, for non-important and important data. 1-replica volumes are also not backed up. 

Some of the data (media collection etc) is mounted as `hostPath` because it is exported via Samba.
I'm looking closely at `Samba at Kubernetes` project and consider trying the samba operator. Then everything can be moved to Longhorn.

### Monitoring
For observability I've installed `kube-prometheus-stack`. It worked out of the box, the only customization needed was to disable KubeScheduler and KubeControllerManager rules - k3s doesn't expose those components.

## Applications
### Smarthome
I run:
- home assistant
- node red
- zigbee2mqtt
- zwavejsui
- openhab
- homebridge

All of it talks to `Mosquitto` MQTT broker.
Also there are two my own scripts - one converts openhab json to zigbee2mqtt json, the other polls the custom weather service. Were converted to Deployment and Cronjob respectively.

For all those apps helm charts are available at `k8s-at-home` project. Those charts use to include "common" library chart. Honestly I didn't like k8s-at-home charts that much - every time common chart is different, they miss sometimes nodeSelector/nodeAffinity (I need it to pin zigbee/zwave pods to the machine with controllers), miss Certificate objects for TLS etc etc. 
Sometimes I just pulled and modified the chart (which is bad), sometimes just added missing objects.

In all cases in `Persistence` sections I used `existingClaim` and created the PVC/PV myself. That was needed to be able to copy data from existing directories before launching k8s instances.

### Calibri
Calibri and Calibri-web do have helm charts at k8s-at-home, without major issues.
Calibri itself is a big issue - it is a QT app in a container with VNC which kinda works with a browser.
For Calibri I wanted to export `Incoming` path via SMB to be able to copy books from my laptop and have Calibri process them. That effectively pinned Calibri to the machine with SMB server running.

To be able to download books from IPFS I've made a kustomization to inject `ipfs/kubo` gateway so that Calibri could access it as localhost:8000.

### ADSB feeder
I run ADSB feeder for FlightRadar24, adsbexchange and some other serivce I don't even remember. It all runs on a RPi with SDR dongle. The setup was done 7 years ago and since that I've logged at that PI only once or twice. there are no containers, just a mess of shell scripts.
This has to be moved to k8s as well but first I need to find out what is running there :)

### Other
There are also some daemons and services - but nothing interesting enough to mention here.

## Tools
* kubectl
* k9s
* jq
* yq
* helm 
* kustomize 
* crictl
* prometheus,grafana 