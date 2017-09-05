# Overlay Networking within Swarm Mode

In the previous guide we explored how a container is able to communicate to the world, through both the hosts interfaces and the docker daemon. 

Now lets look at how Docker Swarm Mode uses overlay networks to allow containers from one node to communicate to containers running on another node. 

In the past blog, we looked at 2 namespaces. The network namespace "Ingress" and the network namespace "Ingress Sandbox". 

Lets see how these are used:

```
[olly@docker0 ~]$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
1858607f95dd        bridge              bridge              local
43hjz9oqpmlt        demonet             overlay             swarm
fddb73392ee6        docker_gwbridge     bridge              local
eca55baa3a88        host                host                local
t3kf9d0orzqx        ingress             overlay             swarm
bf8d0cd1dc2d        none                null                local

[olly@docker0 ~]$ sudo ls -l /var/run/docker/netns
total 0
-r--r--r--. 1 root root 0 Sep  4 14:13 1-43hjz9oqpm
-r--r--r--. 1 root root 0 Sep  4 11:49 1-t3kf9d0orz
-r--r--r--. 1 root root 0 Sep  4 14:13 b4bbd586b39d
-r--r--r--. 1 root root 0 Sep  4 11:49 ingress_sbox
```

Ok firstly lets complete out interface diagram. 

Lets see what other interface are connected to the "Demonet Namespace"

```
[root@docker0 ~]# docker inspect demonet -f {{.Id}}
43hjz9oqpmltxr3tiqb1cvlj9

[root@docker0 ~]# nsenter --net=/var/run/docker/netns/1-43hjz9oqpm ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT 
    link/ether 46:68:08:dc:64:6c brd ff:ff:ff:ff:ff:ff
18: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT 
    link/ether 46:68:08:dc:64:6c brd ff:ff:ff:ff:ff:ff link-netnsid 0
20: veth2@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT 
    link/ether 9a:64:65:68:cb:6a brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

We can see here that there is a a vxlan interface.   
