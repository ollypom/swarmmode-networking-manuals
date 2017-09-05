# Overlay Networking within Swarm Mode

In the previous guide we explored how a container is able to communicate to the world, through both the hosts interfaces and the docker daemon. 

Now lets look at how Docker Swarm Mode uses overlay networks to allow containers from one node to communicate to containers running on another node. 

In the past blog, we ignored 2 network namespaces. 

The one associated with the "Ingress network" and the "ingress_sbox" namespace.

```
[olly@docker0 ~]$ sudo ls -l /var/run/docker/netns
[sudo] password for olly: 
total 0
-r--r--r--. 1 root root 0 Sep  5 10:31 1-t7zfankbs8
-r--r--r--. 1 root root 0 Sep  5 10:29 1-wi6xp5jnpk
-r--r--r--. 1 root root 0 Sep  5 10:31 f9f4ceb4012b
-r--r--r--. 1 root root 0 Sep  5 10:29 ingress_sbox

[olly@docker0 ~]$ docker network ls --filter name=ingress
NETWORK ID          NAME                DRIVER              SCOPE
wi6xp5jnpkzp        ingress             overlay             swarm
```

Lets see what they are, firstly by inspecting the Ingress network.

```
[olly@docker0 ~]$ docker network inspect ingress --format '{{json .Containers}}' | jq
{
  "ingress-sbox": {
    "Name": "ingress-endpoint",
    "EndpointID": "6dff81405a5e045b62ab98a4a394a5d90885629aad8dc397de17401d88d6314a",
    "MacAddress": "02:42:0a:ff:00:02",
    "IPv4Address": "10.255.0.2/16",
    "IPv6Address": ""
  }
}
```

Ooh there is a hidden container called ingress-sbox!!!! This is what is connected to the ingress network. 

Lets take alook at the interfaces for this container.

```
[olly@docker0 ~]$ sudo nsenter --net=/var/run/docker/netns/ingress_sbox ip a
[sudo] password for olly: 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:0a:ff:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.255.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
15: eth1@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
```

Ok, so like any other container in swarm. It has one interface connected to an overlay network (In this case the one called Ingress), and then a second interface connected to the gw bridge. 

This is exactly the same the alpine container, in terms of connections. 

Therfore an updated diagram looks like this.

# IMAGE 3 

Lets see what intefaces are in use when we try and use out alpine container and ping the second alpine container. 


First lets get the IPs of the containers:

```
[olly@docker0 ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
0252f5d8ee9a        alpine:latest       "sleep 50000"       3 hours ago         Up 3 hours                              demoservice.1.dxug6j2lijsr88d9hvugkm8bl
[olly@docker0 ~]$ docker exec 0252f5d8ee9a ip a | grep 192.168.200
    inet 192.168.200.3/24 scope global eth0
    inet 192.168.200.2/32 scope global eth0
```

Host1

```
[olly@docker1 ~]$ docker exec c475cc117d47 ip a | grep 192.168.200
    inet 192.168.200.4/24 scope global eth0
    inet 192.168.200.2/32 scope global eth0
[olly@docker1 ~]$ docker exec c475cc117d47 ping -c 4 192.169.200.3
PING 192.169.200.3 (192.169.200.3): 56 data bytes
64 bytes from 192.169.200.3: seq=0 ttl=52 time=210.701 ms
64 bytes from 192.169.200.3: seq=1 ttl=52 time=224.387 ms
64 bytes from 192.169.200.3: seq=2 ttl=52 time=137.545 ms
64 bytes from 192.169.200.3: seq=3 ttl=52 time=225.336 ms

--- 192.169.200.3 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 137.545/199.492/225.336 ms
```

Now lets ping the container on docker1 from the container on docker0

```
[olly@docker0 ~]$ docker exec 0252f5d8ee9a ping -c 4 192.168.200.4
PING 192.168.200.4 (192.168.200.4): 56 data bytes
64 bytes from 192.168.200.4: seq=0 ttl=64 time=1.493 ms
64 bytes from 192.168.200.4: seq=1 ttl=64 time=2.531 ms
64 bytes from 192.168.200.4: seq=2 ttl=64 time=1.873 ms
64 bytes from 192.168.200.4: seq=3 ttl=64 time=2.314 ms

--- 192.168.200.4 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 1.493/2.052/2.531 ms
```

Ok lets monitor some interfaces to see the traffic flow. 

Terminal 1 we will monitor the veth0 within the demonet namespace. 


Terminal 2 we will monitor the VXLAN interface



# ROUTING MESH!

Ok so we now have an ingress container, attached to an ingress network. This powers the Docker Swarm Mode inbuilt loadbalancer. The Routing Mesh. 

Lets see how it works. First lets clean up our environment. 




The overlay network ingress is created by default when you initialise a docker swarm. A purpose is a catch all, if you start defining services with out the "--network" flag. 

For example:



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
