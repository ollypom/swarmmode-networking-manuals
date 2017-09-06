### Set up a cluster to explore 

This is going to be a very simple environment, to demonstrate the Swarm Mode networking functionality. Therefore to start, I have created a 2 Node Cluster. 1 Manager and 1 Worker. They are running on the latest version of centos 7 and are running the Docker EE 1706 Engine. These nodes are called Docker0.local nad Docker1.local.

```
[olly@docker0 ~]$ docker version
Client:
 Version:      17.03.2-ee-6
 API version:  1.27
 Go version:   go1.7.5
 Git commit:   bdc2646
 Built:        Wed Aug 23 23:08:28 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.2-ee-6
 API version:  1.27 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   bdc2646
 Built:        Wed Aug 23 23:08:28 2017
 OS/Arch:      linux/amd64
 Experimental: false

[olly@docker0 ~]$ uname -r
3.10.0-514.26.2.el7.x86_64

[olly@docker0 ~]$ cat /etc/os-release 
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

My environment is running locally on QEMU / KVM on my laptop. Therefore we have 1 interface per node attached to a local bridge, to allow for node to node communication, and to reach the wide world. 

ens3 is my primary interface and it is on the 192.168.122.0/24 subnet.


```
[olly@docker0 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:d7:3d:cc brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.10/24 brd 192.168.122.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::b99c:7654:5307:a3a6/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:ba:70:52:60 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
```

I have then created a Docker Swarm using the in build Docker Swarm Mode (Swarmkit).

```
[olly@docker0 ~]$ docker swarm init

[olly@docker1 ~]$ docker swarm join --token SWMTKN-1-4fplmikvt0f9u53efbfxtqngf18cl6m3wd9h4ud44bm99qxyxh-8i7lt5kwbawmwapvzqqg48drt 192.168.122.10:2377

[olly@docker0 ~]$ docker node ls
ID                           HOSTNAME       STATUS  AVAILABILITY  MANAGER STATUS
t0k5yhm8g0wft11397n4caz9j *  docker0.local  Ready   Active        Leader
td0ifcvhzy7cexgiixiisnl3b    docker1.local  Ready   Active
``` 

Perfect, we have a cluster.

### Exploring the Namespaces

 Now lets create an overlay network and a service so we can actually start to poke around the various namespaces within SwarmKit.

```
[olly@docker0 ~]$ docker network create -d overlay --subnet 192.168.200.0/24 demonet
t7zfankbs8hm17bi65g0fr4zi
[olly@docker0 ~]$ docker service create --replicas 2 --network demonet --name demoservice alpine sleep 50000
z2p2zg47t2h7qa1g4t5wcvpke

[olly@docker0 ~]$ docker service ps demoservice
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
dxug6j2lijsr        demoservice.1       alpine:latest       docker0.local       Running             Running 32 seconds ago                       
n6ncl5yfszwc        demoservice.2       alpine:latest       docker1.local       Running             Running 32 seconds ago       
```

As you can see we now have a container running on each of my hosts. 

Checking the networks you can see I have my overlay network, and now 4 network namespaces. 

For more information on network namespaces. Head to: https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/

```
[olly@docker0 ~]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
c168bc566c10        bridge              bridge              local
t7zfankbs8hm        demonet             overlay             swarm
90d63100439b        docker_gwbridge     bridge              local
ec08e9c80ede        host                host                local
wi6xp5jnpkzp        ingress             overlay             swarm
f98650512a55        none                null                local

[olly@docker0 ~]$ sudo ls -l /var/run/docker/netns
total 0
-r--r--r--. 1 root root 0 Sep  5 10:31 1-t7zfankbs8
-r--r--r--. 1 root root 0 Sep  5 10:29 1-wi6xp5jnpk
-r--r--r--. 1 root root 0 Sep  5 10:31 f9f4ceb4012b
-r--r--r--. 1 root root 0 Sep  5 10:29 ingress_sbox
```

So what are these namespaces defined within /var/run/docker/netns ?

1. The Alpine Containers Sandbox Namespace. This can be proven by running the inspect command.

```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       4 minutes ago       Up 4 minutes                            demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker inspect 0252f5d8ee9a --format {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/f9f4ceb4012b
```

2. Ingress Overlay. You can see this as the name '1-wi6xp5jnpk' matches the ID of the Ingress Overlay Network

```
[olly@docker0 ~]$ docker network ls --filter name=ingress
NETWORK ID          NAME                DRIVER              SCOPE
wi6xp5jnpkzp        ingress             overlay             swarm
```

3. Demonet Overlay. Following the same logic as the ingress overlay network. The Demonet networks id is t7zfankbs8hm which maps to the namspace '1-t7zfankbs8hm'.

```
[olly@docker0 ~]$ docker network ls --filter name=demonet
NETWORK ID          NAME                DRIVER              SCOPE
t7zfankbs8hm        demonet             overlay             swarm
```

4. Finally we have an Ingress Sandbox network namespace. Ingress is an internal loadbalancer built within the docker network. See the Ingree Guide for more details. 

### How does my container talk to stuff?

As part of the Alpine service we created above, we should now have an alpine container sat there sleeping on my host. I have specified that it should be connected to my "Demonet" network. Lets see what it looks like.

```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       9 minutes ago       Up 9 minutes                            demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker exec 0252f5d8ee9a ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:c0:a8:c8:03 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.3/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.200.2/32 scope global eth0
       valid_lft forever preferred_lft forever
20: eth1@if21: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.3/16 scope global eth1
       valid_lft forever preferred_lft forever
```

Ok some a default container running in Swarm Mode has 2 interfaces + loopback. Out of curiosity you get the same information from running the 'ip a' command within the Sandbox network namespace of the container. We can do this with the 'nsenter' command. 


```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       10 minutes ago      Up 10 minutes                           demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker inspect 0252f5d8ee9a --format {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/f9f4ceb4012b

[olly@docker0 ~]$ sudo nsenter --net=/var/run/docker/netns/f9f4ceb4012b ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:c0:a8:c8:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.200.3/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet 192.168.200.2/32 scope global eth0
       valid_lft forever preferred_lft forever
20: eth1@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.3/16 scope global eth1
       valid_lft forever preferred_lft forever
```

Wow ok, so we can actually see that those network interfaces, and the whole networking stack of the container lives with in that Network Namespace. 

Trying to follow those interfaces we can see how our container talks out.

```
[olly@docker0 ~]$ docker network inspect demonet --format {{.IPAM.Config}}
[{192.168.200.0/24  192.168.200.1 map[]}]

[olly@docker0 ~]$ docker network inspect docker_gwbridge --format {{.IPAM.Config}}
[{172.18.0.0/16  172.18.0.1 map[]}]
```

So eth0 of the container has an address 192.168.200.3, therefore is connected to our demonet overlay network. 

Eth1 on the other hand, has the address 172.18.0.3, therefore is connected to the Network docker_gwbridge. This is the default bridge that lives on my host, that all containers connect to, allowing them to have external communication 

Putting this on a diagram we can see how the container is connect to the outside world. 

From the first diagram we can see that we have 4 namespaces, and the default docker_gwbridge, and my container in sandbox.

![alt text](../master/images/image1.png "Image 1")

Let start working out how we are communicating between the namespaces. This is done using VETH Pairs. So firstly lets see what VETH pairs are within the container's namespace.

```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       18 minutes ago      Up 18 minutes                           demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker inspect 0252f5d8ee9a -f {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/f9f4ceb4012b

[olly@docker0 ~]$ sudo nsenter --net=/var/run/docker/netns/f9f4ceb4012b ip link show
[sudo] password for olly: 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
18: eth0@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT 
    link/ether 02:42:c0:a8:c8:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
20: eth1@if21: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT 
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

Docker nicely names the interfaces so you can understand where they are connected to. 

Interface 18 = eth0. Inteface 18 is connected to Interface 19.

Inteface 20 = eth1. Interface 20 is connected to Interface 21.

From here we want to check that Interface 19 lives in the Demonet Overlay namespace. Allowing container to container communication. 

```
[olly@docker0 ~]$ docker network ls --filter name=demonet
NETWORK ID          NAME                DRIVER              SCOPE
t7zfankbs8hm        demonet             overlay             swarm

[olly@docker0 ~]$ sudo nsenter --net=/var/run/docker/netns/1-t7zfankbs8 ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT 
    link/ether 1a:30:c5:15:42:54 brd ff:ff:ff:ff:ff:ff
17: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT 
    link/ether 1a:30:c5:15:42:54 brd ff:ff:ff:ff:ff:ff link-netnsid 0
19: veth0@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT 
    link/ether ca:29:3c:e5:9e:8f brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

Perfect we can see inteface 19 lives in the demonet overlay namespace. As well as a Network Bridge. A VXLAN interface (more on this in the next blog), and a loopback. 

For completeleness lets check that Interface 21 is living on the Docker Host itself. 

```
[olly@docker0 ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:d7:3d:cc brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.10/24 brd 192.168.122.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::b99c:7654:5307:a3a6/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:ba:70:52:60 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
8: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:fa:6d:94:b5 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 scope global docker_gwbridge
       valid_lft forever preferred_lft forever
    inet6 fe80::42:faff:fe6d:94b5/64 scope link 
       valid_lft forever preferred_lft forever
16: veth1a1ccf0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP 
    link/ether 2e:db:f1:a9:b0:d0 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::2cdb:f1ff:fea9:b0d0/64 scope link 
       valid_lft forever preferred_lft forever
21: veth0c4d246@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP 
    link/ether 26:df:a4:02:5f:4c brd ff:ff:ff:ff:ff:ff link-netnsid 3
    inet6 fe80::24df:a4ff:fe02:5f4c/64 scope link 
       valid_lft forever preferred_lft forever
```

Perfect there is interface 21. So we can now update our diagram accordingly. 

![alt text](../master/images/image2.png "Image 2")

### Traffic routing

Just for completeness here I lets see how the container knows where to send the traffic.

```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       27 minutes ago      Up 27 minutes                           demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker exec 0252f5d8ee9a ip route show
default via 172.18.0.1 dev eth1 
172.18.0.0/16 dev eth1  src 172.18.0.3 
192.168.200.0/24 dev eth0  src 192.168.200.3 
```

Ok nice so there are some static routes in play. The default option is to send the traffic out of the eth1 to the internet. However if we are talking on the demonet subnet of 192.168.200.0/24 then go down eth0. 

We can even monitor the veths in the sandbox namespace, to prove this is working correctly. Here I am going to open up a second terminal session to Docker Host 'docker0'.

Terminal1:
```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       28 minutes ago      Up 28 minutes                           demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker exec 0252f5d8ee9a apk add --update tcpdump

[olly@docker0 ~]$ docker exec 0252f5d8ee9a tcpdump -i eth0 -vv
```

Terminal2:
```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       29 minutes ago      Up 29 minutes                           demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker exec 0252f5d8ee9a ping 192.168.200.1 -c 4
PING 192.168.200.1 (192.168.200.1): 56 data bytes
64 bytes from 192.168.200.1: seq=0 ttl=64 time=0.085 ms
64 bytes from 192.168.200.1: seq=1 ttl=64 time=0.301 ms
64 bytes from 192.168.200.1: seq=2 ttl=64 time=1.079 ms
64 bytes from 192.168.200.1: seq=3 ttl=64 time=0.297 ms

--- 192.168.200.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.085/0.440/1.079 ms
```

Now on terminal 1 you should see:


Terminal1:
```
[olly@docker0 ~]$ docker exec 0252f5d8ee9a tcpdump -i eth0 -vv
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
10:01:00.800598 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.200.1 tell 0252f5d8ee9a, length 28
10:01:00.800621 ARP, Ethernet (len 6), IPv4 (len 4), Reply 192.168.200.1 is-at 1a:30:c5:15:42:54 (oui Unknown), length 28
10:01:00.800623 IP (tos 0x0, ttl 64, id 17253, offset 0, flags [DF], proto ICMP (1), length 84)
    0252f5d8ee9a > 192.168.200.1: ICMP echo request, id 5888, seq 0, length 64
10:01:00.800642 IP (tos 0x0, ttl 64, id 57, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.200.1 > 0252f5d8ee9a: ICMP echo reply, id 5888, seq 0, length 64
10:01:01.800947 IP (tos 0x0, ttl 64, id 17379, offset 0, flags [DF], proto ICMP (1), length 84)
    0252f5d8ee9a > 192.168.200.1: ICMP echo request, id 5888, seq 1, length 64
10:01:01.801024 IP (tos 0x0, ttl 64, id 793, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.200.1 > 0252f5d8ee9a: ICMP echo reply, id 5888, seq 1, length 64
10:01:02.802422 IP (tos 0x0, ttl 64, id 17764, offset 0, flags [DF], proto ICMP (1), length 84)
    0252f5d8ee9a > 192.168.200.1: ICMP echo request, id 5888, seq 2, length 64
10:01:02.802494 IP (tos 0x0, ttl 
```

Ok awesome so if I am sending a request to the gateway or a container on that DemoNet overlay network, it is going to the Demonet overlay namespace via eth0. Awesome :)

We can double check to make sure traffic is going to the global namespace when it needs to go outside. 

Terminal1
```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       31 minutes ago      Up 31 minutes                           demoservice.1.dxug6j2lijsr88d9hvugkm8bl

[olly@docker0 ~]$ docker exec 0252f5d8ee9a tcpdump -i eth1 -vv
```

Terminal2
```
[olly@docker0 ~]$ docker exec 0252f5d8ee9a ping 8.8.8.8 -c 4
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=61 time=6.976 ms
64 bytes from 8.8.8.8: seq=1 ttl=61 time=5.929 ms
64 bytes from 8.8.8.8: seq=2 ttl=61 time=5.770 ms
64 bytes from 8.8.8.8: seq=3 ttl=61 time=5.287 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 5.287/5.990/6.976 ms
```

And once again on Terminal 1 we should now have a few packets :)

Terminal1
```
[olly@docker0 ~]$ docker exec 0252f5d8ee9a tcpdump -i eth1 -vv
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
10:03:53.865755 IP (tos 0x0, ttl 64, id 62720, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 8960, seq 0, length 64
10:03:53.872569 IP (tos 0x0, ttl 57, id 50076, offset 0, flags [none], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 8960, seq 0, length 64
10:03:54.400640 IP (tos 0x0, ttl 64, id 47849, offset 0, flags [DF], proto UDP (17), length 66)
    172.18.0.3.33602 > google-public-dns-a.google.com.53: [bad udp cksum 0xbc64 -> 0x3446!] 29314+ PTR? 8.8.8.8.in-addr.arpa. (38)
10:03:54.507849 IP (tos 0x0, ttl 57, id 43925, offset 0, flags [none], proto UDP (17), length 110)
    google-public-dns-a.google.com.53 > 172.18.0.3.33602: [udp sum ok] 29314 q: PTR? 8.8.8.8.in-addr.arpa. 1/0/0 8.8.8.8.in-addr.arpa. PTR google-public-dns-a.google.com. (82)
10:03:54.510280 IP (tos 0x0, ttl 64, id 47942, offset 0, flags [DF], proto UDP (17), length 69)
    172.18.0.3.38744 > google-public-dns-a.google.com.53: [bad udp cksum 0xbc67 -> 0x79bf!] 17719+ PTR? 3.0.18.172.in-addr.arpa. (41)
10:03:54.516941 IP (tos 0x0, ttl 57, id 11062, offset 0, flags [none], proto UDP (17), length 69)
    google-public-dns-a.google.com.53 > 172.18.0.3.38744: [udp sum ok] 17719 NXDomain q: PTR? 3.0.18.172.in-addr.arpa. 0/0/0 (41)
10:03:54.866345 IP (tos 0x0, ttl 64, id 63489, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 8960, seq 1, length 64
10:03:54.871385 IP (tos 0x0, ttl 57, id 50515, offset 0, flags [none], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 8960, seq 1, length 64
10:03:55.866955 IP (tos 0x0, ttl 64, id 63761, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 8960, seq 2, length 64
10:03:55.873911 IP (tos 0x0, ttl 57, id 51220, offset 0, flags [none], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 8960, seq 2, length 64
10:03:56.867490 IP (tos 0x0, ttl 64, id 64265, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.3 > google-public-dns-a.google.com: ICMP echo request, id 8960, seq 3, length 64
10:03:56.874199 IP (tos 0x0, ttl 57, id 51699, offset 0, flags [none], proto ICMP (1), length 84)
    google-public-dns-a.google.com > 172.18.0.3: ICMP echo reply, id 8960, seq 3, length 64
10:03:58.877301 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 172.18.0.1 tell 172.18.0.3, length 28
10:03:58.877333 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 172.18.0.3 tell 172.18.0.1, length 28
10:03:58.877391 ARP, Ethernet (len 6), IPv4 (len 4), Reply 172.18.0.3 is-at 02:42:ac:12:00:03 (oui Unknown), length 28
10:03:58.877376 ARP, Ethernet (len 6), IPv4 (len 4), Reply 172.18.0.1 is-at 02:42:fa:6d:94:b5 (oui Unknown), length 28
10:03:59.407728 IP (tos 0x0, ttl 64, id 49516, offset 0, flags [DF], proto UDP (17), length 69)
    172.18.0.3.36703 > google-public-dns-a.google.com.53: [bad udp cksum 0xbc67 -> 0x3b40!] 35761+ PTR? 1.0.18.172.in-addr.arpa. (41)
10:03:59.412610 IP 
```

Awesome. Happy days. This shows the isolation for docker container overlay networking. If required you can actually run containers which don't have a leg in the global namespace at all. Completely isolated. 

The 2nd blog post here will show how the ingress networking, routing and loadbalancing works. 
