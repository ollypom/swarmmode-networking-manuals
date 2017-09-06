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

Terminal 1 - Ping from the 1st alpine container to the second alpine container

```
[olly@docker0 ~]$ docker exec e060b5eeb1ca ping 192.168.200.3 -c 1
```

Terminal 2 - Monitor the VXLAN (VTEP) interface in the network namespace
```
[olly@docker0 ~]$ sudo nsenter --net=/var/run/docker/netns/1-t7zfankbs8 tcpdump -i vxlan0 -vv
tcpdump: WARNING: vxlan0: no IPv4 address assigned
tcpdump: listening on vxlan0, link-type EN10MB (Ethernet), capture size 65535 bytes
13:05:23.167902 IP (tos 0x0, ttl 64, id 55630, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.200.5 > 192.168.200.3: ICMP echo request, id 5120, seq 0, length 64
13:05:23.168463 IP (tos 0x0, ttl 64, id 45497, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.200.3 > 192.168.200.5: ICMP echo reply, id 5120, seq 0, length 64
13:05:28.173017 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.200.3 tell 192.168.200.5, length 28
13:05:28.173082 ARP, Ethernet (len 6), IPv4 (len 4), Reply 192.168.200.3 is-at 02:42:c0:a8:c8:03 (oui Unknown), length 28
```

Terminal 3 - Monitor the external interface, in the host
```
[olly@docker0 ~]$ sudo tcpdump -i ens3 port '4789' -vv
tcpdump: listening on ens3, link-type EN10MB (Ethernet), capture size 65535 bytes

13:08:49.009543 IP (tos 0x0, ttl 64, id 36664, offset 0, flags [none], proto UDP (17), length 134)
    docker0.local.40984 > 192.168.122.11.4789: [no cksum] VXLAN, flags [I] (0x08), vni 4097
IP (tos 0x0, ttl 64, id 23591, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.200.5 > 192.168.200.3: ICMP echo request, id 6144, seq 0, length 64

13:08:49.009969 IP (tos 0x0, ttl 64, id 22374, offset 0, flags [none], proto UDP (17), length 134)
    192.168.122.11.58719 > docker0.local.4789: [no cksum] VXLAN, flags [I] (0x08), vni 4097
IP (tos 0x0, ttl 64, id 50994, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.200.3 > 192.168.200.5: ICMP echo reply, id 6144, seq 0, length 64
```

We can see here that using the requests are going from the container, down the VETH into the overlay network namespace. We are then creating a tunnel through from the VTEP (VXLAN0) to the second host.

To prove that this is going on we can see that in the terminal 3 output, that traffic is going to the IP of the second host 192.168.122.11. And is using the default VXLAN interface 4789. 

# Image 4

More details on VXLAN can be found here:

http://blog.nigelpoulton.com/demystifying-docker-overlay-networking/

https://vincent.bernat.im/en/blog/2017-vxlan-linux

# ROUTING MESH!

Ok so we now have an ingress container, attached to an ingress network. This powers the Docker Swarm Mode inbuilt loadbalancer. The Routing Mesh. 

Lets see how it works. First lets clean up our environment, as we are going to use a nginx service this time. 

```
[olly@docker0 ~]$ docker service rm demoservice
demoservice

[olly@docker0 ~]$ docker network rm demonet
demonet

[olly@docker0 ~]$ docker network create -d overlay --subnet 192.168.201.0/24 nginxnet
k9gwqu0r4v7aw0vzc2rwo0dxr

[olly@docker0 ~]$ docker service create --replicas 2 --network nginxnet --name nginxservice --publish 80:80 nginx
twhqav6amkmcdp8a3bbxysmfw
```

Ok so we have remove the alpine containers, and created an nginx service, on the nginxnet subnet and have publish the port 80. Therefore from out host we should be able to test this from my workstation.

```
olly@xps ~> curl 192.168.122.10:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Ok lets see how the requests comes in. Our first stop is the NATing Table of the host

```
[olly@docker0 ~]$ sudo iptables -nvL -t nat
Chain PREROUTING (policy ACCEPT 148 packets, 10140 bytes)
 pkts bytes target     prot opt in     out     source               destination         
  136  8260 DOCKER-INGRESS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
  337 20542 DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 134 packets, 8140 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 203 packets, 13393 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER-INGRESS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 205 packets, 13513 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      docker_gwbridge  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match src-type LOCAL
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
    0     0 MASQUERADE  all  --  *      !docker_gwbridge  172.18.0.0/16        0.0.0.0/0           

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
    0     0 RETURN     all  --  docker_gwbridge *       0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-INGRESS (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    2   120 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:172.18.0.2:80
  134  8140 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0  
```

We can we see here that the second to last line, says, any traffic heading to Port 80 (Our publish port) redirect it to 172.18.0.2.

That 172.18.0.0/24 subnet belongs to our Docker Gateway Bridge:

```
[olly@docker0 ~]$ ip  add show dev docker_gwbridge
3: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ec:2d:ff:62 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 scope global docker_gwbridge
       valid_lft forever preferred_lft forever
    inet6 fe80::42:ecff:fe2d:ff62/64 scope link 
       valid_lft forever preferred_lft forever
```

If we have a look in side of the Ingress Sandbox hidden container. We can see that the IP Address one of its veth pairs is 172.18.0.3 :)

```
[olly@docker0 ~]$ sudo nsenter --net=/var/run/docker/netns/ingress_sbox ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
38: eth0@if39: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:0a:ff:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.255.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
40: eth1@if41: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
```

So we can see where the traffic goes.

No once in this container we need to do loadbalance between requests, using IPTables and Service Discovery discussed in the final article.
