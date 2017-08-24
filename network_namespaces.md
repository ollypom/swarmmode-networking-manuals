# Network Namespaces and Beyond

Based on these blogs for swarm classic:

http://techblog.d2-si.eu/2017/04/25/deep-dive-into-docker-overlay-networks-part-1.html

It felt like right to make one for Swarm Mode :)

TODO - Missed the early steps where I evaluated the Ingress Overlay Network and the Ingress Sandbox Namespace.

### Set up a cluster to explore 

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
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:ae:58:89:00  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

docker_gwbridge: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:e3ff:fe97:caa4  prefixlen 64  scopeid 0x20<link>
        ether 02:42:e3:97:ca:a4  txqueuelen 0  (Ethernet)
        RX packets 14312  bytes 1532922 (1.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 14312  bytes 1532922 (1.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

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
[olly@docker0 ~]$ docker swarm init --advertise-addr enp0s3

[olly@docker1 ~]$ docker swarm join --advertise-addr enp0s3  \
    --token SWMTKN-1-4bgrvcqunq6oye74v3cjve2jbyn1t9u7mnaek3rw2clbl5van0-14dgns2w0phm1koy5yom2tzgf \
    192.168.10.6:2377

[olly@docker0 ~]$ docker node ls
ID                           HOSTNAME       STATUS  AVAILABILITY  MANAGER STATUS
t0k5yhm8g0wft11397n4caz9j *  docker0.local  Ready   Active        Leader
td0ifcvhzy7cexgiixiisnl3b    docker1.local  Ready   Active
``` 

Perfect.

### Exploring the Namespaces

 Now lets create an overlay network and a service so we can actually start to poke around the various namespaces within SwarmKit.

```
$ docker network create -d overlay --subnet 192.168.200.0/24 demonet
$ docker service create --replicas 2 --network demonet --name demoservice alpine sleep 10000

[olly@docker0 ~]$ docker service ps demoservice
ID            NAME               IMAGE          NODE           DESIRED STATE  CURRENT STATE          ERROR                             PORTS
zazcrztrp8no  demoservice.1      alpine:latest  docker0.local  Running        Running 8 minutes ago
lbg6y0b0y3zh   \_ demoservice.1  alpine:latest  docker0.local  Shutdown       Failed 8 minutes ago   "starting container failed: Ad…"
qfbik6vam8q6  demoservice.2      alpine:latest  docker1.local  Running        Running 8 minutes ago
```

As you can see we now have a container running on each of my hosts. 

Checking the networks you can see I have my overlay network, and now 4 network namespaces. 

```
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
```

So what are these namespaces defined within /var/run/docker/netns ?

1. The Alpine Containers Sandbox Namespace. This can be proven by running the inspect command.

```
[olly@docker0 ~]$ docker inspect 43f05d7a2438 | grep Sandbox
            "SandboxID": "f30f9eab7ed4e2e637806da7e006b743ac2b9bf8aac2a0ff2843eb02ef7c94a9",
            "SandboxKey": "/var/run/docker/netns/f30f9eab7ed4",
```

2. Ingress Overlay. You can see this as the name '1-jk79gn3pi7' matches the ID of the Ingress Overlay Network

```
[olly@docker0 ~]$ docker network ls --filter name=ingress
NETWORK ID          NAME                DRIVER              SCOPE
jk79gn3pi7ge        ingress             overlay             swarm
```

3. Demonet Overlay. Following the same logic as the ingress overlay network. The Demonet network is contained within its own namespace.

4. Finally we have an Ingress Sandbox. TODO Work out what this does?

### How does my container talk to stuff?

As part of the Alpine service we created above, we should now have an alpine container sat there sleeping on my host. I have specified that it should be connected to my "Demonet" network. Lets see what it looks like.

```
[olly@docker0 ~]# docker ps
CONTAINER ID        IMAGE                                                                            COMMAND             CREATED             STATUS              PORTS               NAMES
9e79d66b5112        alpine@sha256:1072e499f3f655a032e88542330cf75b02e7bdf673278f701d7ba61629ee3ebe   "sleep 10000"       15 minutes ago      Up 14 minutes                           demoservice.1.zazcrztrp8noilfyignita3tf


[olly@docker0 ~]$ docker exec 9e79d66b5112 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:c0:a8:c8:03
          inet addr:192.168.200.3  Bcast:0.0.0.0  Mask:255.255.255.0
          inet6 addr: fe80::42:c0ff:fea8:c803/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:41 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:3254 (3.1 KiB)  TX bytes:648 (648.0 B)

eth1      Link encap:Ethernet  HWaddr 02:42:ac:12:00:03
          inet addr:172.18.0.3  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe12:3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:42 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:3304 (3.2 KiB)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

We can actually do the same thing within the Sandbox namespace of the container, using the nsenter command. 

```
[root@docker0 olly]# docker inspect 9e79d66b5112 -f {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/06e0e24ce06d

[root@docker0 olly]# nsenter --net=/var/run/docker/netns/06e0e24ce06d ifconfig
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
```

From the ifconfig out put we can see that the container has 2 interfaces. Eth0 and Eth1. Looking at the IP address we can see the container has 1 leg in the Demonet Overlay network. And 1 leg connected to the Docker_GWBridge which lives in the global namespace.

```
[olly@docker0 ~]$ docker network inspect demonet -f '{{json .IPAM.Config}}' | jq '.'
[
  {
    "Subnet": "192.168.200.0/24",
    "Gateway": "192.168.200.1"
  }
]

[olly@docker0 ~]$ docker network inspect docker_gwbridge -f '{{json .IPAM.Config}}' | jq '.'
[
  {
    "Subnet": "172.18.0.0/16",
    "Gateway": "172.18.0.1"
  }
]
```

I think its time to start drawing diagrams to work out how that is connected to the outside world.

From the first diagram we can see that we have 4 namespaces, and the default docker_gwbridge, and my container in sandbox.

**Image 1**

Let start working out how we are communicating between the namespaces. This is done using VETH Pairs. So firstly lets see what VETH pairs are within the container's namespace.

```
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/06e0e24ce06d ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT
    link/ether 02:42:c0:a8:c8:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
20: eth1@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

Ok so: eth0 = interface 18 and is connected to interface 19. eth1 = interface 20 and is connected to interface 21.

So lets go and make sure that interface 19 lives inside of the demonet overlay namespace. 

```
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/1-jv8c30bcmn ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT
    link/ether 1a:34:cf:83:2c:11 brd ff:ff:ff:ff:ff:ff
17: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT
    link/ether f6:1c:e9:9a:60:bd brd ff:ff:ff:ff:ff:ff link-netnsid 0
19: veth2@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT
    link/ether 1a:34:cf:83:2c:11 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

Perfect. There is interface 19.

Now lets go and see if interface 21 is in the global namespace connected to the Docker_GWBridge. 

```
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
```

Perfect there is interface 21. So we can now update our diagram accordingly. 

** Image 2 **

Just for completeness here I lets see how the container knows where to send the traffic.

```
[root@docker0 olly]# docker exec 3e53de6af276 ip route show
default via 172.18.0.1 dev eth1
172.18.0.0/16 dev eth1  src 172.18.0.3
192.168.200.0/24 dev eth0  src 192.168.200.4
```

Ok nice so there are some static routes in play. The default option is to send the traffic out of the eth1 to the internet. However if we are talking on the demonet subnet of 192.168.200.0/24 then go down eth0. 

We can even monitor the veths in the sandbox namespace, to prove this is working correctly. Here I am going to open up terminal sessions.

Terminal1:
```
[root@docker0 ~]# docker exec 3e53de6af276 apk add --update tcpdump
[root@docker0 ~]# docker exec 3e53de6af276 tcpdump -i eth0 -vv
```

Terminal2:
```
[olly@docker0 ~]$ docker exec 3e53de6af276 ping 192.168.200.1 -c 4
PING 192.168.200.1 (192.168.200.1): 56 data bytes
64 bytes from 192.168.200.1: seq=0 ttl=64 time=0.139 ms
64 bytes from 192.168.200.1: seq=1 ttl=64 time=0.144 ms
64 bytes from 192.168.200.1: seq=2 ttl=64 time=0.110 ms
64 bytes from 192.168.200.1: seq=3 ttl=64 time=0.110 ms

--- 192.168.200.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.110/0.125/0.144 ms
```

Terminal1:
```
[root@docker0 ~]# docker exec 3e53de6af276 tcpdump -i eth0 -vv
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:50:24.396714 IP (tos 0x0, ttl 64, id 19031, offset 0, flags [DF], proto ICMP (1), length 84)
    3e53de6af276 > 192.168.200.1: ICMP echo request, id 18176, seq 0, length 64
15:50:24.396783 IP (tos 0x0, ttl 64, id 55889, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.200.1 > 3e53de6af276: ICMP echo reply, id 18176, seq 0, length 64
15:50:25.399797 IP (tos 0x0, ttl 64, id 19915, offset 0, flags [DF], proto ICMP (1), length 84)
    3e53de6af276 > 192.168.200.1: ICMP echo request, id 18176, seq 1, length 64
15:50:25.399847 IP (tos 0x0, ttl 64, id 56855, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.200.1 > 3e53de6af276: ICMP echo reply, id 18176, seq 1, length 64
15:50:26.404153 IP (tos 0x0, ttl 64, id 20855, offset 0, flags [DF], proto ICMP (1), length 84)
    3e53de6af276 > 192.168.200.1: ICMP echo request, id 18176, seq 2, length 64
15:50:26.404188 IP (tos 0x0, ttl 64, id 57649, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.200.1 > 3e53de6af276: ICMP echo reply, id 18176, seq 2, length 64
15:50:27.409870 IP (tos 0x0, ttl 64, id 21257, offset 0, flags [DF^C
```

Ok awesome so if I am sending a request to the gateway or a container on that DemoNet overlay network, it is going to the Demonet overlay namespace. Awesome :)

We can double check to make sure traffic is going to the global namespace when it needs to go outside. 

Terminal1
```
[root@docker0 ~]# docker exec 3e53de6af276 tcpdump -i eth1 -vv
```

Terminal2
```
[olly@docker0 ~]$ docker exec 3e53de6af276 ping 8.8.8.8 -c 4
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=61 time=6.976 ms
64 bytes from 8.8.8.8: seq=1 ttl=61 time=5.929 ms
64 bytes from 8.8.8.8: seq=2 ttl=61 time=5.770 ms
64 bytes from 8.8.8.8: seq=3 ttl=61 time=5.287 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 5.287/5.990/6.976 ms
```

Terminal1
```
[root@docker0 ~]# docker exec 3e53de6af276 tcpdump -i eth1 -vv
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
15:53:17.623446 IP (tos 0x0, ttl 64, id 8514, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 20224, seq 0, length 64
15:53:17.629460 IP (tos 0x0, ttl 61, id 31464, offset 0, flags [DF], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 20224, seq 0, length 64
15:53:18.040380 IP (tos 0x0, ttl 64, id 47549, offset 0, flags [DF], proto UDP (17), length 66)
    172.18.0.3.36804 > max-az-2k12-dc1.internal.maximus-it.com.53: [bad udp cksum 0x5869 -> 0x2fd9!] 52840+ PTR? 8.8.8.8.in-addr.arpa. (38)
15:53:18.056098 IP (tos 0x0, ttl 63, id 31465, offset 0, flags [none], proto UDP (17), length 110)
    max-az-2k12-dc1.internal.maximus-it.com.53 > 172.18.0.3.36804: [udp sum ok] 52840 q: PTR? 8.8.8.8.in-addr.arpa. 1/0/0 8.8.8.8.in-addr.arpa. PTR google-public-dns-a.google.com. (82)
15:53:18.057096 IP (tos 0x0, ttl 64, id 47552, offset 0, flags [DF], proto UDP (17), length 69)
    172.18.0.3.45495 > max-az-2k12-dc1.internal.maximus-it.com.53: [bad udp cksum 0x586c -> 0x3d0b!] 52103+ PTR? 3.0.18.172.in-addr.arpa. (41)
15:53:18.086138 IP (tos 0x0, ttl 63, id 31466, offset 0, flags [none], proto UDP (17), length 69)
    max-az-2k12-dc1.internal.maximus-it.com.53 > 172.18.0.3.45495: [udp sum ok] 52103 NXDomain q: PTR? 3.0.18.172.in-addr.arpa. 0/0/0 (41)
15:53:18.625708 IP (tos 0x0, ttl 64, id 8617, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 20224, seq 1, length 64
15:53:18.631365 IP (tos 0x0, ttl 61, id 31467, offset 0, flags [DF], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 20224, seq 1, length 64
15:53:19.042464 IP (tos 0x0, ttl 64, id 48393, offset 0, flags [DF], proto UDP (17), length 69)
    172.18.0.3.33958 > max-az-2k12-dc1.internal.maximus-it.com.53: [bad udp cksum 0x586c -> 0x3348!] 1115+ PTR? 4.0.16.172.in-addr.arpa. (41)
15:53:19.060691 IP (tos 0x0, ttl 63, id 31468, offset 0, flags [none], proto UDP (17), length 122)
    max-az-2k12-dc1.internal.maximus-it.com.53 > 172.18.0.3.33958: [udp sum ok] 1115* q: PTR? 4.0.16.172.in-addr.arpa. 1/0/0 4.0.16.172.in-addr.arpa. PTR max-az-2k12-dc1.internal.maximus-it.com. (94)
15:53:19.629414 IP (tos 0x0, ttl 64, id 9188, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 20224, seq 2, length 64
15:53:19.634906 IP (tos 0x0, ttl 61, id 31469, offset 0, flags [DF], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 20224, seq 2, length 64
15:53:20.629957 IP (tos 0x0, ttl 64, id 9281, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 20224, seq 3, length 64
15:53:20.635036 IP (tos 0x0, ttl 61, id 31470, offset 0, flags [DF], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 20224, seq 3, length 64
15:53:22.629378 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 172.18.0.1 tell 172.18.0.3, length 28
```

Awesome. Happy days. This shows the isolation for docker container overlay networking. If required you can actually run containers which don't have a leg in the global namespace at all. Completely isolated. 

TODO - Show how the ingress network is connected. And however Ingress Overlay networking works for multiple hosts.

```
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
```
