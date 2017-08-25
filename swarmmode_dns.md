# Now lets look abit more about DNS queries.

From docker 1.10 onwards, we now include a DNS server with the Docker Daemon. All containers /etc/hosts are configured to point this internal DNS server, and then when requests come in to go externally from the docker environment we use the Variables defined in the Hosts /etc/resolv.conf or take valuables that are defined within the /etc/docker/daemon.conf. 

For more info see:

https://docs.docker.com/engine/userguide/networking/configure-dns/

Lets double check that this is running and accessible within my environment. 

Ok lets see what the Alpine Containers DNS configuration is.

``` 
[root@docker0 ~]# docker exec 9e79d66b5112 cat /etc/resolv.conf
search internal.maximus-it.com local
nameserver 127.0.0.11
options ndots:0
```

Ok it is set up to use the embedded DNS server, running on a local address. Hmmm. Lets query this DNS server, by doing a lookup against the Service we have created.

```
[root@docker0 ~]# docker exec 9e79d66b5112 apk add --update bind-tools

[root@docker0 ~]# docker exec 9e79d66b5112 dig demoservice.demonet

; <<>> DiG 9.11.1-P1 <<>> demoservice.demonet
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21676
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;demoservice.demonet.		IN	A

;; ANSWER SECTION:
demoservice.demonet.	600	IN	A	192.168.200.2

;; Query time: 1 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Wed Aug 23 11:07:12 UTC 2017
;; MSG SIZE  rcvd: 72
```

Ok perfect, the dns server has responded and told us that the Docker Service is accessible on its VIP address. Great I can now use that address within my applications for service to service integration.

However lets look at this a level down using the network namespace tools we have played about with previously...

Firstly lets have a look at how we are accessing this DNS service, by monitoring a containers interface during a DNS query. In my head the first place to look, is to see if the DNS server is running in the Namespace of the Overlay Network. 

Terminal1
```
[root@docker0 ~]# docker exec 9e79d66b5112 apk add --update tcpdump
[root@docker0 ~]# docker exec 9e79d66b5112 tcpdump -i eth0 port 53
```

Terminal2
```
[olly@docker0 ~]$ docker exec 9e79d66b5112 nslookup demoservice.demonet
Server:		127.0.0.11
Address:	127.0.0.11#53

Non-authoritative answer:
Name:	demoservice.demonet
Address: 192.168.200.2
```

Terminal1
```
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/06e0e24ce06d  tcpdump -i eth0 port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
^C
```

Interesting ok, so it doesn't access the DNS server in the Demonet Overlay network.

As its a 127.0.0.11 address, lets monitor the loopback interface during a query to see if that brings up any clues. 

Terminal1
```
[root@docker0 ~]# docker exec 9e79d66b5112 apk add --update tcpdump
[root@docker0 ~]# docker exec 9e79d66b5112 tcpdump -i lo port 53
```

Terminal 2
```
[olly@docker0 ~]$ docker exec 9e79d66b5112 nslookup demoservice.demonet
Server:		127.0.0.11
Address:	127.0.0.11#53

Non-authoritative answer:
Name:	demoservice.demonet
Address: 192.168.200.2
```

Terminal 1
```
[root@docker0 ~]# nsenter --net=/var/run/docker/netns/06e0e24ce06d  tcpdump -i lo -v port 53
tcpdump: listening on lo, link-type EN10MB (Ethernet), capture size 65535 bytes
13:00:32.012763 IP (tos 0x0, ttl 64, id 56840, offset 0, flags [DF], proto UDP (17), length 100)
    127.0.0.11.domain > localhost.50551: 38911 1/0/0 demoservice.demonet. A 192.168.200.2 (72)
13:00:32.016087 IP (tos 0x0, ttl 64, id 56841, offset 0, flags [DF], proto UDP (17), length 65)
    127.0.0.11.domain > localhost.39780: 39653 0/0/0 (37)
```

Ahhhh ok. Nice. So during a DNS lookup we are using the loopback interface. The interesting thing here though is. "127.0.0.11.domain > localhost.50551" "127.0.0.11.domain > localhost.39780". It is looking like we are redirecting our traffic awaway from port 53. The default DNS port. 

Well lets see how we are doing this. Lets check some NATing rules in the Sandbox Namespace of the Container.

```
[root@docker0 ~] docker inspect 9e79d66b5112 -f {{.NetworkSettings.SandboxKey}}
/var/run/docker/netns/4ba56594526d

[root@docker0 ~]# nsenter --net=/var/run/docker/netns/4ba56594526d iptables -nvL -t nat
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 5 packets, 360 bytes)
 pkts bytes target     prot opt in     out     source               destination
    2   137 DOCKER_OUTPUT  all  --  *      *       0.0.0.0/0            127.0.0.11
    0     0 DNAT       icmp --  *      *       0.0.0.0/0            192.168.200.2        icmptype 8 to:127.0.0.1

Chain POSTROUTING (policy ACCEPT 7 packets, 497 bytes)
 pkts bytes target     prot opt in     out     source               destination
    2   137 DOCKER_POSTROUTING  all  --  *      *       0.0.0.0/0            127.0.0.11
    0     0 SNAT       all  --  *      *       0.0.0.0/0            192.168.200.0/24     ipvs to:192.168.200.3

Chain DOCKER_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            127.0.0.11           tcp dpt:53 to:127.0.0.11:39923
    2   137 DNAT       udp  --  *      *       0.0.0.0/0            127.0.0.11           udp dpt:53 to:127.0.0.11:58175

Chain DOCKER_POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 SNAT       tcp  --  *      *       127.0.0.11           0.0.0.0/0            tcp spt:39923 to::53
    0     0 SNAT       udp  --  *      *       127.0.0.11           0.0.0.0/0            udp spt:58175 to::53
```


So we can see that there is an IPTables rule to move traffic heading to :53 to either a 39923 or a 58175 port on localhost depending on whether this is UDP or TCP. If you run a net stat for open ports, you can see in fact this forwards to the PID of  the docker daemon.

[root@docker0 ~]# nsenter --net=/var/run/docker/netns/4ba56594526d netstat -c -tunlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.11:39923        0.0.0.0:*               LISTEN      988/dockerd
udp        0      0 127.0.0.11:58175        0.0.0.0:*                           988/dockerd

And yep, you can do service discovery against these ports. 

[olly@docker0 ~]$ docker exec 2036107e55ce dig @127.0.0.11 -p 58175 demoservice.demonet

; <<>> DiG 9.11.1-P1 <<>> @127.0.0.11 -p 58175 demoservice.demonet
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62564
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;demoservice.demonet.		IN	A

;; ANSWER SECTION:
demoservice.demonet.	600	IN	A	192.168.200.2

;; Query time: 0 msec
;; SERVER: 127.0.0.11#58175(127.0.0.11)
;; WHEN: Wed Aug 23 14:35:08 UTC 2017
;; MSG SIZE  rcvd: 72



