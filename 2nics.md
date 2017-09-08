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

Hmm i'm not sure why this is available on both Interfaces. 

Lets see whats going on here.

```
olly@swarm1:~$ docker network create -d overlay --subnet 192.168.100.0/24 testernet

olly@swarm1:~$ docker service create --replicas 2 --network testernet --name testernginx nginx 

olly@swarm1:~$ docker service update --publish-add 80:80 testernginx
```

Ok lets see if our nxginx service is only accessible on the data path port

```
olly@xps ~/D/d/m/swarm-mode-manuals> curl http://10.0.0.10                                                                                                                                             130 master?
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
olly@xps ~/D/d/m/swarm-mode-manuals> curl http://10.1.0.10                                                                                                                                                 master?
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

Ballls its not!


