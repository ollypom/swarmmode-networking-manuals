### Swarm Mode Service Discovery 

In the last guide we looked at how to find the VIPs of our service. In swarm mode we that use the VIP to balance between multiple tasks (containers) that sit behind that service. 

Lets see how we can find out, once again playing in the Container Sandbox network namespace.

Firstly lets use the Docker CLI to get the VIP's endpoint.

```
[root@docker0 olly]# docker service inspect -f '{{json .Endpoint.VirtualIPs}}' demoservice | jq
[
  {
    "NetworkID": "jv8c30bcmnzak9qcmu4sbeajw",
    "Addr": "192.168.200.2/24"
  }
]
```

Ok, so here we can see the VIP, which is the last page we were able to get the DNS record for. Docker SwarmKit uses Linux IPVS Loadbalancing to distrbute load from this VIP to the tasks.

https://github.com/docker/libnetwork/blob/6a0dfa192942a71da066a4ba157eac7cf208deae/ipvs/ipvs.go

IPVS "Is a lightweight Transport-Layer loadbalancer inside the Linux kernal"

http://www.linuxvirtualserver.org/software/ipvs.html

You can check the IPVS is installed and set up correctly with:

```
[root@docker0 olly]# wget https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh
[root@docker0 olly]# chmod +x check-config.sh
[root@docker0 olly]# ./check-config.sh  | grep IP_VS
warning: /proc/config.gz does not exist, searching other paths for kernel config ...
- CONFIG_IP_VS: enabled (as module)
- CONFIG_IP_VS_NFCT: enabled
- CONFIG_IP_VS_RR: enabled (as module)
```

If we take a look inside of the Container Sandbox Namespace we should be able to see IPVS configuration. 

```
[root@docker0 olly]# docker inspect ed41f1f74e28 -f {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/2c34d17c8cde
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/2c34d17c8cde iptables -nvL -t mangle
Chain PREROUTING (policy ACCEPT 14 packets, 1039 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 14 packets, 1039 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 16 packets, 1215 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  *      *       0.0.0.0/0            192.168.200.2        MARK set 0x102

Chain POSTROUTING (policy ACCEPT 16 packets, 1215 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

Looking within the IP Tables Mangle table we can set that we have Marked traffic heading to the VIP. The mangle table is where we can modify entries in the packet headers.

Looking at the above MARK, we can see we are in fact marking within the packet header 0x102 (or 258 in decimal)

Now lets see what the 258 responds to in IPVS.

For this we will need the ipvsadm package, and then run this with in the Container Sandbox Namespace

```
[root@docker0 ~]# yum install -y ipvsadm
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/4ba56594526d ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  258 rr
  -> docker0.local:0              Masq    1      0          0
  -> 192.168.200.4:0              Masq    1      0          0
```

Ahh nice. Ok we can see that in this case we have an entry on FWM 258 or firewall mark 258 or firwall mark 0x102 in hex :D

And we can see we have 2 entries. One is the localhost, i.e. the container in which you are running the command. And 1 is the remote container running on the second host. 

And if I increase the number of replicas within my service, you can see that more entries appear in this table. 

```
[root@docker0 ~]# docker service update --replicas 5 demoservice
demoservice

[root@docker0 ~]# docker service ls
ID            NAME         MODE        REPLICAS  IMAGE
inlooqo4gsdp  demoservice  replicated  5/5       alpine:latest
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/4ba56594526d ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  258 rr
  -> docker0.local:0              Masq    1      0          0
  -> 192.168.200.4:0              Masq    1      0          0
  -> 192.168.200.5:0              Masq    1      0          0
  -> 192.168.200.6:0              Masq    1      0          0
  -> 192.168.200.7:0              Masq    1      0          0
```

Awesome you can now see that I have 5 entries within my LVS table. Happy Days. 

If i add a new service in to my cluster. Watch how it also now getting entries within the IPVS table

Are you sure that IP Virtual Server is built in the kernel or as module?
[olly@docker0 ~]$ docker service create --replicas 2 --network demonet --name demoservice2 alpine sleep 10000
e6lx2ib10eoutri6kh10clnsq

[root@docker0 olly]#  nsenter --net=/var/run/docker/netns/2c34d17c8cde iptables -nvL -t mangle
Chain PREROUTING (policy ACCEPT 402 packets, 34072 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 402 packets, 34072 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 402 packets, 33720 bytes)
 pkts bytes target     prot opt in     out     source               destination
 5654  475K MARK       all  --  *      *       0.0.0.0/0            192.168.200.8        MARK set 0x100
    0     0 MARK       all  --  *      *       0.0.0.0/0            192.168.200.2        MARK set 0x102

Chain POSTROUTING (policy ACCEPT 402 packets, 33720 bytes)
 pkts bytes target     prot opt in     out     source               destination

[root@docker0 ~]#  nsenter --net=/var/run/docker/netns/2c34d17c8cde ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  256 rr
  -> 192.168.200.9:0              Masq    1      0          0
  -> 192.168.200.10:0              Masq    1      0          0
FWM  258 rr
  -> docker0.local:0              Masq    1      0          0
  -> 192.168.200.4:0              Masq    1      0          0

Awesome we can see that the Mangle tables have been updated and that a new rr entry has been set up in the load balancer.

### What happens if we create our service with the DNS Round Robin flag, rather than using the default Virtual IP? 

https://docs.docker.com/engine/swarm/networking/#configure-service-discovery

```
[root@docker0 olly]# docker service create --replicas 2 --network demonet --name demoservice3 --endpoint-mode dnsrr alpine sleep 5000
0n7m16x4hu472gxyt65f9ugsi

[root@docker0 olly]# docker service inspect -f '{{json .Spec.EndpointSpec}}' 0n7m16x4hu472gxyt65f9ugsi
{"Mode":"dnsrr"}
```

Ok the "--endpoint-mode dnsrr" has configured out dns round robin. Lets see what this looks like in the mangle and ipvsadm tables.

```
[root@docker0 olly]# docker inspect f2a63fcbfaa5 -f {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/99a93f2bd549
[root@docker0 olly]# nsenter --net=/var/run/docker/netns/99a93f2bd549  iptables -nvL -t mangle
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  *      *       0.0.0.0/0            192.168.200.2        MARK set 0x100
    0     0 MARK       all  --  *      *       0.0.0.0/0            192.168.200.5        MARK set 0x102

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

[root@docker0 olly]# nsenter --net=/var/run/docker/netns/99a93f2bd549 ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  256 rr
  -> 192.168.200.3:0              Masq    1      0          0
  -> 192.168.200.4:0              Masq    1      0          0
FWM  258 rr
  -> 192.168.200.6:0              Masq    1      0          0
  -> 192.168.200.7:0              Masq    1      0          0
```

Hmmm nothing. No VIP and No round robin load balancer. As expected. 

Lets see what happens if we talk to the DNS server.

```
[root@docker0 olly]# docker exec f2a63fcbfaa5 apk --update add bind-tools
fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.6/community/x86_64/APKINDEX.tar.gz
(1/4) Installing libgcc (6.3.0-r4)
(2/4) Installing libxml2 (2.9.4-r4)
(3/4) Installing bind-libs (9.11.1_p1-r1)
(4/4) Installing bind-tools (9.11.1_p1-r1)
Executing busybox-1.26.2-r5.trigger
OK: 9 MiB in 15 packages

[root@docker0 olly]# docker exec f2a63fcbfaa5 dig demoservice3.demonet

; <<>> DiG 9.11.1-P1 <<>> demoservice3.demonet
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 1182
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;demoservice3.demonet.		IN	A

;; ANSWER SECTION:
demoservice3.demonet.	600	IN	A	192.168.200.9
demoservice3.demonet.	600	IN	A	192.168.200.8

;; Query time: 0 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Fri Aug 25 13:20:23 UTC 2017
;; MSG SIZE  rcvd: 110
```

Ahh here we go we have 2 A entries in for the DNS record. Instead DNS will resolve to a container randomly. 

A simple ping test can prove that it will automatically balance requests between tasks

```
^C
[olly@docker0 ~]$ docker exec ef4c804323d5 ping demoservice3.demonet -c 4
PING demoservice3.demonet (192.168.200.9): 56 data bytes
64 bytes from 192.168.200.9: seq=0 ttl=64 time=0.728 ms
64 bytes from 192.168.200.9: seq=1 ttl=64 time=0.524 ms
64 bytes from 192.168.200.9: seq=2 ttl=64 time=1.014 ms
64 bytes from 192.168.200.9: seq=3 ttl=64 time=0.771 ms

--- demoservice3.demonet ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.524/0.759/1.014 ms
[olly@docker0 ~]$ docker exec ef4c804323d5 ping demoservice3.demonet -c 4
PING demoservice3.demonet (192.168.200.8): 56 data bytes
64 bytes from 192.168.200.8: seq=0 ttl=64 time=0.052 ms
64 bytes from 192.168.200.8: seq=1 ttl=64 time=0.152 ms
64 bytes from 192.168.200.8: seq=2 ttl=64 time=0.086 ms
64 bytes from 192.168.200.8: seq=3 ttl=64 time=0.092 ms

--- demoservice3.demonet ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.052/0.095/0.152 ms
```

** THis is how it works on Docker for Windows / Docker EE Windows Server Nodes. We don't implement the VIP as we don't have a Linux IPVS in there, we have to rely on DNS round robin. **

All of the steps above should be shown in the following slide:

![alt text](../master/images/sdslide.png "SD Slide")

