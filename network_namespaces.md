# Network Namespaces and Beyond

Based on these blogs for swarm classic:

http://techblog.d2-si.eu/2017/04/25/deep-dive-into-docker-overlay-networks-part-1.html

http://techblog.d2-si.eu/2017/04/25/deep-dive-into-docker-overlay-networks-part-2.html

http://techblog.d2-si.eu/2017/04/25/deep-dive-into-docker-overlay-networks-part-3.html

It felt like right to make one for Swarm Mode :)

TODO - Missed the early steps where I evaluated the Ingress Overlay Network and the Ingress Sandbox Namespace.

## Set up a cluster to explore 

I have just created a 2 node cluster. 1 Manager and 1 Worker. They are running on the latest version of centos 7 and are running the Docker EE 1706 Engine. These nodes are called Docker0.local nad Docker1.local.

```
[root@docker0 ~]# docker version
Client:
 Version:      17.03.2-ee-5
 API version:  1.27
 Go version:   go1.7.5
 Git commit:   fa09039
 Built:        Thu Jul 20 00:18:48 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.2-ee-5
 API version:  1.27 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   fa09039
 Built:        Thu Jul 20 00:18:48 2017
 OS/Arch:      linux/amd64
 Experimental: false

[root@docker0 ~]# uname -r
3.10.0-514.26.2.el7.x86_64
[root@docker0 ~]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```

These nodes are running on Virtual Box with 1 NIC on a Host-Only network on the 192.168.10.x/24 subnet. And 1 NIC nated to the host with a 10.x.x.x address.

```
[root@docker0 ~]# ifconfig
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.10.6  netmask 255.255.255.0  broadcast 192.168.10.255
        inet6 fe80::a00:27ff:fe35:8335  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:35:83:35  txqueuelen 1000  (Ethernet)
        RX packets 41148  bytes 5107966 (4.8 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 37493  bytes 8483555 (8.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.3.15  netmask 255.255.255.0  broadcast 10.0.3.255
        inet6 fe80::3dcd:f55f:9714:e624  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:b3:03:4a  txqueuelen 1000  (Ethernet)
        RX packets 82314  bytes 107956881 (102.9 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 38830  bytes 2368524 (2.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 12222  bytes 1320282 (1.2 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12222  bytes 1320282 (1.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

I have then created a Docker Swarm using the in build Docker Swarm Mode (Swarmkit).

```



$ docker network create -d overlay --subnet 192.168.200.0/24 demonet
$ docker service create --replicas 2 --network demonet --name demoservice alpine sleep 10000

[olly@docker0 ~]$ docker service ps demoservice
ID            NAME               IMAGE          NODE           DESIRED STATE  CURRENT STATE          ERROR                             PORTS
zazcrztrp8no  demoservice.1      alpine:latest  docker0.local  Running        Running 8 minutes ago
lbg6y0b0y3zh   \_ demoservice.1  alpine:latest  docker0.local  Shutdown       Failed 8 minutes ago   "starting container failed: Ad…"
qfbik6vam8q6  demoservice.2      alpine:latest  docker1.local  Running        Running 8 minutes ago

We should have a container running on each of my hosts. Checking the networks you can see I have my overlay network, and now 4 network namespaces. 

1. An Alpine Container
2. Ingress Overlay
3. Demonet Overlay 
4. Ingress Sandbox

[olly@docker0 ~]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
1586a330769e        bridge              bridge              local
jv8c30bcmnza        demonet             overlay             swarm
ae5a99b47a8e        docker_gwbridge     bridge              local
1f54d786801e        host                host                local
jk79gn3pi7ge        ingress             overlay             swarm
684885c1baa0        none                null                local

[root@docker0 olly]# ls -l /var/run/docker/netns
total 0
-r--r--r--. 1 root root 0 Aug 23 11:11 06e0e24ce06d
-r--r--r--. 1 root root 0 Aug 23 09:22 1-jk79gn3pi7
-r--r--r--. 1 root root 0 Aug 23 11:11 1-jv8c30bcmn
-r--r--r--. 1 root root 0 Aug 23 09:22 ingress_sbox

Connecting to the Container Network Namespace

[root@docker0 olly]# docker ps
CONTAINER ID        IMAGE                                                                            COMMAND             CREATED             STATUS              PORTS               NAMES
9e79d66b5112        alpine@sha256:1072e499f3f655a032e88542330cf75b02e7bdf673278f701d7ba61629ee3ebe   "sleep 10000"       15 minutes ago      Up 14 minutes                           demoservice.1.zazcrztrp8noilfyignita3tf

[root@docker0 olly]# docker inspect 9e79d66b5112 -f {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/06e0e24ce06d

[root@docker0 olly]# nsenter --net=$C0netns ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 192.168.200.3  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::42:c0ff:fea8:c803  prefixlen 64  scopeid 0x20<link>
        ether 02:42:c0:a8:c8:03  txqueuelen 0  (Ethernet)
        RX packets 14  bytes 1128 (1.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 648 (648.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.3  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:acff:fe12:3  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ac:12:00:03  txqueuelen 0  (Ethernet)
        RX packets 8  bytes 648 (648.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 648 (648.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

So we can see that it has VETHs in 2 namespaces. The Demonet Overlay and the Ingress Overlay. 

We can see the veth IDs to see what they are connected too..

[root@docker0 ~]# nsenter --net=/var/run/docker/netns/06e0e24ce06d ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT
    link/ether 02:42:c0:a8:c8:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
20: eth1@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1

[root@docker0 olly]# nsenter --net=$C0netns  ethtool -S eth0
NIC statistics:
     peer_ifindex: 19
[root@docker0 olly]# nsenter --net=$C0netns  ethtool -S eth1
NIC statistics:
     peer_ifindex: 21

So eth0 is in the Namespace of the Demonet Overlay network:

[root@docker0 ~]# nsenter --net=/var/run/docker/netns/1-jv8c30bcmn ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT
    link/ether 1a:34:cf:83:2c:11 brd ff:ff:ff:ff:ff:ff
17: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT
    link/ether f6:1c:e9:9a:60:bd brd ff:ff:ff:ff:ff:ff link-netnsid 0
19: veth2@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT
    link/ether 1a:34:cf:83:2c:11 brd ff:ff:ff:ff:ff:ff link-netnsid 1

And eth1 is in the Global Namespace 

[root@docker0 ~]# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 08:00:27:35:83:35 brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 08:00:27:b3:03:4a brd ff:ff:ff:ff:ff:ff
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT
    link/ether 02:42:ca:14:bb:9a brd ff:ff:ff:ff:ff:ff
5: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
    link/ether 02:42:0c:99:9e:cd brd ff:ff:ff:ff:ff:ff
11: vethb0f3583@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT
    link/ether ee:a6:c3:12:df:41 brd ff:ff:ff:ff:ff:ff link-netnsid 1
21: veth90d0481@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT
    link/ether fe:02:c8:af:60:f5 brd ff:ff:ff:ff:ff:ff link-netnsid 4
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/1-jv8c30bcmn ip -d link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0 addrgenmode eui64
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT
    link/ether 1a:34:cf:83:2c:11 brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge forward_delay 1500 hello_time 200 max_age 2000 addrgenmode eui64
17: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT
    link/ether f6:1c:e9:9a:60:bd brd ff:ff:ff:ff:ff:ff link-netnsid 0 promiscuity 1
    vxlan id 4097 srcport 0 0 dstport 4789 proxy l2miss l3miss ageing 300
    bridge_slave addrgenmode eui64
19: veth2@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT
    link/ether 1a:34:cf:83:2c:11 brd ff:ff:ff:ff:ff:ff link-netnsid 1 promiscuity 1
    veth
    bridge_slave addrgenmode eui64

