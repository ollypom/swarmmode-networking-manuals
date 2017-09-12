# Swarm Mode using seperate interfaces for Control Plane and Data Traffic

In this environment I have the following setup:

```
2 Networks:

Network1: 10.0.0.0/24
Network2: 10.1.0.0/24

4 VMs:

swarm1.local ens3 10.0.0.10 ens9 10.1.0.10
swarm2.local ens3 10.0.0.11 ens9 10.1.0.11
swarm3.local ens3 10.0.0.12 ens9 10.1.0.12
swarm4.local ens3 10.0.0.13 ens9 10.1.0.13
```

I then set up my environment with the following comands:

```
olly@swarm1:~$ docker swarm init --advertise-addr ens9 --data-path-addr ens3
Swarm initialized: current node (lid8i9hwho4a0a5lcha6kbrvf) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0fiiep16xbpxm5mijmy390lrcvp1wd943r6nhsc0g14c5a3xh9-1ct6803na4ryty49fwt92spxy 10.1.0.10:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

olly@swarm2:~$ docker swarm join --token SWMTKN-1-0fiiep16xbpxm5mijmy390lrcvp1wd943r6nhsc0g14c5a3xh9-1ct6803na4ryty49fwt92spxy --advertise-addr ens9 --data-path-addr ens3 10.1.0.10:2377
olly@swarm3:~$ docker swarm join --token SWMTKN-1-0fiiep16xbpxm5mijmy390lrcvp1wd943r6nhsc0g14c5a3xh9-1ct6803na4ryty49fwt92spxy --advertise-addr ens9 --data-path-addr ens3 10.1.0.10:2377
olly@swarm4:~$ docker swarm join --token SWMTKN-1-0fiiep16xbpxm5mijmy390lrcvp1wd943r6nhsc0g14c5a3xh9-1ct6803na4ryty49fwt92spxy --advertise-addr ens9 --data-path-addr ens3 10.1.0.10:2377
```

Swarm looks like its deployed ok. Lets now install Docker UCP on to of it :)

```
olly@swarm1:~$ docker container run --rm -it --name ucp   -v /var/run/docker.sock:/var/run/docker.sock   docker/ucp:2.2.2 install   --host-address ens9   --interactive
...
INFO[0049] Login to UCP at https://10.1.0.10:443        
INFO[0049] Username: admin                              
INFO[0049] Password: (your admin password)  
```

Looks ok. Interesting this service is accessible on the "Addvertise Addr" interface. Not the Data Path interface.

Testing this:

```
olly@xps ~/D/d/m/swarm-mode-manuals> curl -k https://10.0.0.10                                                                                                                                             master?
<!doctype html>
<html lang="en">
<head>
...
</html>

olly@xps ~/D/d/m/swarm-mode-manuals> curl -k https://10.1.0.10                                                                                                                                             master?
<!doctype html>
<html lang="en">
...
</html>
```

Ok so published ports is not affected by the split in interfaces. 

Instead the 2 things that are is the Control Plan of Swarm and the Data Plan of Swarm. 

The differences between these layers:

![alt text](../master/images/networkplanes.png "Network Planes Slide")

Therefore within in my environment we can summarise this as:

ens3 10.0.0.0/24 will carry all the "Data Plane" traffic. All Container to Container communication accross overlay networks will go accross this network. 

ens9 10.1.0.0/24 will carry all the "Control Plane" traffic. There all swarm Gossip and Service Discovery will go accross this network. 

Lets prove this my inspect traffic on an interface. 

Looking at the Docker EE UCP Requirements https://docs.docker.com/datacenter/ucp/2.2/guides/admin/install/system-requirements/#ports-used we can see the following ports are required for these planes:

Control Plane:
 - 2366 Classic Swarm
 - 2377 Swarm Mode - Cluster Mgmt and Raft
 - 7946 Swarm Mode - Gossip 

Data Plane:
 - 4789 Overlay Networking Between Nodes

### Control Plane

Classic Swarm 2376:
```
olly@swarm1:~$ sudo timeout 2 tcpdump -vvv -i ens9 port 2376
tcpdump: listening on ens9, link-type EN10MB (Ethernet), capture size 262144 bytes
11:49:33.946165 IP (tos 0x0, ttl 63, id 36498, offset 0, flags [DF], proto TCP (6), length 746)
    10.1.0.10.2376 > 10.1.0.12.51358: Flags [P.], cksum 0x16f4 (incorrect -> 0x628f), seq 84266628:84267322, ack 2514489048, win 283, options [nop,nop,TS val 1856967 ecr 1855407], length 694
<trimmed>
```

Swarm Mode 2377 (RAFT):
```
olly@swarm1:~$ sudo timeout 2 tcpdump -vvv -i ens9 port 2377
tcpdump: listening on ens9, link-type EN10MB (Ethernet), capture size 262144 bytes
11:54:30.213088 IP (tos 0x0, ttl 64, id 21257, offset 0, flags [DF], proto TCP (6), length 170)
    10.1.0.10.39644 > 10.1.0.11.2377: Flags [P.], cksum 0x14b3 (incorrect -> 0x9eb5), seq 2476465720:2476465838, ack 778518783, win 241, options [nop,nop,TS val 1931033 ecr 1930158], length 118
<trimmed>
```

Swarm Mode 7946 (Gossip):
```
olly@swarm1:~$ sudo timeout 2 tcpdump -vvv -i ens9 port 7946
tcpdump: listening on ens9, link-type EN10MB (Ethernet), capture size 262144 bytes
11:55:21.006691 IP (tos 0x0, ttl 64, id 43985, offset 0, flags [DF], proto UDP (17), length 99)
    10.1.0.10.7946 > 10.1.0.13.7946: [bad udp cksum 0x1479 -> 0x05b3!] UDP, length 71
11:55:21.008214 IP (tos 0x0, ttl 64, id 18668, offset 0, flags [DF], proto UDP (17), length 77)
    10.1.0.13.7946 > 10.1.0.10.7946: [bad udp cksum 0x1463 -> 0x6c13!] UDP, length 49
```

### Data Plane

Here we will swap interfaces, instead of monitoring traffic on ens9, we will now monitor traffic on ens3.

VXLAN 4789 (Overlay Networking):
```
olly@swarm1:~$ docker network create -d overlay --subnet 192.168.100.0/24 testernet

olly@swarm1:~$ docker service create --replicas 2 --network testernet --name testernginx --publish 80:80 nginx 

olly@swarm1:~$ docker service create --replicas 1 --network testernet --name testeralpine alpine sleep 5000
```

Terminal1:
```
olly@swarm1:~$ sudo timeout 2 tcpdump -vvv -i ens3 port 4789
```

Terminal2:
```
olly@swarm1:~$ docker exec <container-id> apk --update add curl
olly@swarm1:~$ docker exec <container-id> curl http://testernginx.testernet
```

Termina1:
```
olly@swarm1:~$ sudo timeout 2 tcpdump -vvv -i ens3 port 4789
tcpdump: listening on ens9, link-type EN10MB (Ethernet), capture size 262144 bytes
11:55:21.006691 IP (tos 0x0, ttl 64, id 43985, offset 0, flags [DF], proto UDP (17), length 99)
    10.1.0.10.7946 > 10.1.0.13.7946: [bad udp cksum 0x1479 -> 0x05b3!] UDP, length 71
11:55:21.008214 IP (tos 0x0, ttl 64, id 18668, offset 0, flags [DF], proto UDP (17), length 77)
    10.1.0.13.7946 > 10.1.0.10.7946: [bad udp cksum 0x1463 -> 0x6c13!] UDP, length 49
```


Awesome we can now see that we have seperated traffic
