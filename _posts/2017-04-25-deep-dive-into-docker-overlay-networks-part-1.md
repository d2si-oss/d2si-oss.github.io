---
author: Laurent Bernaille
layout: post
title: "Deep dive into Docker Overlay Networks: Part 1"
twitter: lbernail
keywords: docker overlay network,overlay network,vxlan,netlink,dockercon,linux networking
---

## Introduction
At [D2SI](http://d2-si.eu), we have been using Docker since
its very beginning and have been helping many projects go into production.
We believe that going into production requires a
strong understanding of the technology to be able to debug complex issues,
analyze unexpected behaviors or troubleshoot performance degradations. That is
why we have tried to understand as best as we can the technical components used
by Docker.

This blog post is focused on the Docker network overlays. The Docker network
overlay driver relies on several technologies: network namespaces, VXLAN,
Netlink and a distributed key-value store. This article will present each of
these mechanisms one by one along with their userland tools and show hands-on
how they interact together when setting up an overlay to connect containers.

This post is derived from the presentation I gave at
[DockerCon2017](http://2017.dockercon.com/) in Austin. The slides are available
[here](https://www.slideshare.net/lbernail/deep-dive-in-docker-overlay-networks).

All the code used in this post is available on
[GitHub](https://github.com/lbernail/dockercon2017).

## Docker Overlay Networks
First, we are going to build an overlay network between Docker hosts. In our
example, we will do this with three hosts: two running Docker and one running
Consul. Docker will use Consul to store the overlay networks metadata that needs
to be shared by all the Docker engines: container IPs, MAC addresses and
location.  Before Docker 1.12, Docker required an external Key-Value store (Etcd
or Consul) to create overlay networks and Docker Swarms (now often referred to
as "classic Swarm"). Starting with Docker 1.12, Docker can now rely on an
internal Key-Value store to create Swarms and overlay networks ("Swarm mode" or
"new swarm"). We chose to use Consul because it allows us to look into the keys
stored by Docker and understand better the role of the Key-Value store. We are
running Consul on a single node but in a real environment we would need a
cluster of at least three nodes for resiliency.

In our example, the servers will have the following IP addresses:
- consul: 10.0.0.5
- docker0: 10.0.0.10
- docker1: 10.0.0.0.11

<img src="/assets/2017-04-25-deep-dive-into-docker-overlay-networks-part-1/servers-setup.png" alt="Servers setup" width="250" style="margin: 0px auto;display:block;" />

### Starting the Consul and Docker services
The first thing we need to do is to start a Consul server. To do this, we simply
download Consul from [here](https://www.consul.io). We can then start a very
minimal Consul service with the following command:

```bash
consul agent -server -dev -ui -client 0.0.0.0
```

We use the following flags:
- server: start the consul agent in server mode
- dev: create a standalone Consul server without any persistency
- ui: start a small web interface allowing us to easily look at the keys stored
  by Docker and their values
- client 0.0.0.0: bind all network interfaces for client access (default is
  127.0.0.1 only)

To configure the Docker engines to use Consul as an Key-Value store, we start
the daemons with the cluster-store option:

```bash
dockerd -H fd:// --cluster-store=consul://consul:8500 --cluster-advertise=eth0:2376
```

The cluster-advertise option specifies which IP to advertise in the cluster for
a docker host (this option is not optional). This command assumes that consul
resolves to 10.0.0.5 in our case.

If we look at the at the Consul UI, we can see that Docker created some keys,
but the network key: http://consul:8500/v1/kv/docker/network/v1.0/network/ is
still empty.

<img src="/assets/2017-04-25-deep-dive-into-docker-overlay-networks-part-1/consul-start.png" width="600" alt="Consul content" style="margin: 0px auto;display:block;" />

You can easily create the same environment in AWS using the terraform setup in
the [GitHub](https://github.com/lbernail/dockercon2017) repository. All the
default configuration (in particular the region to use) is in variables.tf. You
will need to give a value to the key_pair variable, either using the command
line (terraform apply -var key_pair=demo) or by modifying the variables.tf file.
The three instances are configured with userdata: consul and docker are
installed and started with the good options, an entry is added to /etc/hosts so
consul resolves into the IP address of the consul server. When connecting to
consul or docker servers, you should use the public IP addresses (given in
terraform outputs).

### Creating an Overlay
We can now create an overlay network between our two Docker nodes:

```console
docker0:~$ docker network create --driver overlay --subnet 192.168.0.0/24 demonet
620dd594834293e912bc17931d589c41bac318734d1084632f02da3177708bdc
```

We are using the overlay driver, and are choosing 192.168.0.0/24 as a subnet for
the overlay (this parameter is optional but we want to have addresses very
different from the ones on the hosts to simplify the analysis).

Let's check that we configured our overlay correctly by listing networks on both
hosts.

```console
NETWORK ID          NAME                DRIVER              SCOPE
eb096cb816c0        bridge              bridge              local
620dd5948342        demonet             overlay             global
d538d58b17e7        host                host                local
f2ee470bb968        none                null                local

docker1:~$ docker network ls
admin@docker1:~$ docker network ls
docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
eb7a05eba815        bridge              bridge              local
620dd5948342        demonet             overlay             global
4346f6c422b2        host                host                local
5e8ac997ecfa        none                null                local
```

This looks good: both Docker nodes know the demonet network and it has the same
id (*620dd5948342*) on both hosts.

Let's now check that our overlay works by creating a container on docker0 and
trying to ping it from docker1. On docker0, we create a C0 container, attach it
to our overlay, explicitly give it an IP address (192.168.0.100) and make it
sleep. On docker1 we create a container attached to the overlay network and
running a ping command targeting C0.

```console
docker0:~$ docker run -d --ip 192.168.0.100 --net demonet --name C0 debian sleep 3600

docker1:~$ docker run -it --rm --net demonet debian bash
root@e37bf5e35f83:/# ping 192.168.0.100
PING 192.168.0.100 (192.168.0.100): 56 data bytes
64 bytes from 192.168.0.100: icmp_seq=0 ttl=64 time=0.618 ms
64 bytes from 192.168.0.100: icmp_seq=1 ttl=64 time=0.483 ms
```

We can see that the connectivity between both containers is OK. If we try to
ping C0 from docker1, it does not work because docker1 does not know anything
about 192.168.0.0/24 which is isolated in the overlay.

```console
docker1:~$ ping 192.168.0.100
PING 192.168.0.100 (192.168.0.100) 56(84) bytes of data.
^C--- 192.168.0.100 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3024ms
```

Here is what we have built so far:

<img src="/assets/2017-04-25-deep-dive-into-docker-overlay-networks-part-1/first-overlay.png" alt="First overlay" width="600" style="margin: 0px auto;display:block;" />

## Under the hood
Now that we have built an overlay let's try and see what makes it work.

### Network configuration of the containers
What is the network configuration of C0 on docker0? We can exec into the
container to find out:

```console
docker0:~$ docker exec C0 ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:c0:a8:00:64 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.100/24 scope global eth0
       valid_lft forever preferred_lft forever
9: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
```

We have two interfaces (and the loopback) in the container:
- eth0: configured with an IP in the 192.168.0.0/24 range. This interface is the
  one in our overlay.
- eth1: configured with an IP in 172.18.0.2/16 range, which we did not
  configure anywhere

What about the routing configuration?

```console
docker0:~$ docker exec C0 ip route show
default via 172.18.0.1 dev eth1
172.18.0.0/16 dev eth1  proto kernel  scope link  src 172.18.0.2
192.168.0.0/24 dev eth0  proto kernel  scope link  src 192.168.0.100
```

The routing configuration indicates that the default route is via eth1, which
means that this interface can be used to access resources outside of the
overlay. We can verify this easily by pinging an external IP address.

```console
$ docker exec -it C0 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=51 time=0.957 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=51 time=0.975 ms
```
Note that it is possible to create an overlay where containers do not have access to
external networks using the ```--internal``` flag.

Let's see if we can get more information on these interfaces:

```console
docker0:~$ docker exec C0 ip -details link show eth0
6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:c0:a8:00:64 brd ff:ff:ff:ff:ff:ff promiscuity 0
    veth

docker0:~$ docker exec C0 ip -details link show eth1
9: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff promiscuity 0
    veth
```

The type of both interfaces is *veth*. veth interfaces always always come in
pair connected with a virtual wire. The two peered veth can be in different
network namespaces which allows traffic to move from one namespace to another.
These two veth are used to get outside of the container network namespace.

Here is what we have found out so far:

<img src="/assets/2017-04-25-deep-dive-into-docker-overlay-networks-part-1/container-interfaces.png" alt="Container interfaces" width="600" style="margin: 0px auto;display:block;" />

We now need to identify the interfaces peered with each veth.

### What is the container connected to?
We can identify the other end of a veth using the ethtool command. However this
command is not available in our container. We can execute this command inside
our container using "nsenter" which allows us to enter one or several namespaces
associated with a process or using "ip netns exec" which relies on iproute to
execute a command in a given network namespace. In our examples, we will use ip
netns.

The first thing we need to do is to identify the network namespace of the
container. We can achieve this by inspecting the container, and extracting what
we need from the SandboxKey:

{% raw %}
    docker0:~$ docker inspect C0 -f {{.NetworkSettings.SandboxKey}}
    /var/run/docker/netns/e4b8ecb7ae7c

    docker0:~$ sandbox=$(docker inspect C0 -f {{.NetworkSettings.SandboxKey}})
    docker0:~$ C0netns=${sandbox##*/}
    docker0:~$ echo $C0netns
    e4b8ecb7ae7c
{% endraw %}

Docker does not create symlinks in the /var/run/netns directory which is where
ip netns is looking for network namespaces. To solve this, we simply need to add
a symlink (if you use the terraform setup this symlink is already present).

```bash
sudo ln -s /var/run/docker/netns /var/run/netns
```

We can now run ip netns commands. For instance if we want to list network
namespaces:

```console
docker0:~$ sudo ip netns ls
e4b8ecb7ae7c
1-620dd59483
```

We can also execute host commands inside the network namespace of a container
(even if this container does not have the command):

```console
docker0:~$ sudo ip netns exec $C0netns ip addr show eth0
6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:c0:a8:00:64 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.100/24 scope global eth0
       valid_lft forever preferred_lft forever
```

Let's see what are the interface indexes associated with the peers of eth0 and
eth1:

```console
docker0:~$ sudo ip netns exec $C0netns ethtool -S eth0
NIC statistics:
    peer_ifindex: 7
docker0:~$ sudo ip netns exec $C0netns ethtool -S eth1
NIC statistics:
    peer_ifindex: 10
```

Using nsenter, we could execute the same commands:

{% raw %}
    docker0:~$ sandbox=$(docker inspect C0 -f {{.NetworkSettings.SandboxKey}})
    docker0:~$ sudo nsenter --net=$sandbox ip addr show eth0
    6: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
        link/ether 02:42:c0:a8:00:64 brd ff:ff:ff:ff:ff:ff
        inet 192.168.0.100/24 scope global eth0
           valid_lft forever preferred_lft forever
{% endraw %}

We are now looking for interfaces with indexes 7 and 10. We can first look on
the host itself:

```console
docker0:~$ ip -details link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 promiscuity 0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 06:e2:c0:20:ec:9f brd ff:ff:ff:ff:ff:ff promiscuity 0
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:a7:17:99:39 brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge
8: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:be:d6:b0:c5 brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge
10: vethbc521fc: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP mode DEFAULT group default
    link/ether 32:a1:47:1a:7f:1e brd ff:ff:ff:ff:ff:ff promiscuity 1
    veth
    bridge_slave
```

We can see from this output that we have no trace of interface 7 but we have
found interface 10, the peer of eth1. In addition, this interface is plugged on
a bridge called "docker_gwbridge". What is this bridge? If we list the networks
managed by docker, we can see that it has appeared in the list:

```console
docker0:~$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
eb096cb816c0        bridge              bridge              local
620dd5948342        demonet             overlay             global
f6823b311fd2        docker_gwbridge     bridge              local
d538d58b17e7        host                host                local
f2ee470bb968        none                null                local
```

We can now inspect it:

```console
docker0:~$ docker inspect docker_gwbridge
"Name": "docker_gwbridge",
"Driver": "bridge",
"IPAM": {
    "Driver": "default",
    "Options": null,
    "Config": [
        {
            "Subnet": "172.18.0.0/16",
            "Gateway": "172.18.0.1"
        }
    ]
},
"Options": {
    "com.docker.network.bridge.enable_icc": "false",
    "com.docker.network.bridge.enable_ip_masquerade": "true",
    "com.docker.network.bridge.name": "docker_gwbridge"
 }
```

I removed part of the output to focus on the essential pieces of information:
- this network uses the driver bridge (the same one used by the standard docker
  bridge, docker0)
- it uses subnet 172.18.0.0/16, which is consistent with eth1
- enable_icc is set to false which means we cannot use this bridge for
  inter-container communication
- enable_ip_masquerade is set to true, which means the traffic from the
  container will be NATed to access external networks (which we saw earlier when
  we successfully pinged 8.8.8.8)

We can verify that inter-container communication is disabled by trying to ping
C0 on its eth1 address (172.18.0.2) from another container on docker0 also
attached to demonet:

```console
docker0:~$ docker run --rm -it --net demonet debian ping 172.18.0.2
PING 172.18.0.2 (172.18.0.2): 56 data bytes
^C--- 172.18.0.2 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```

Here is an updated view of what we have found:

<img src="/assets/2017-04-25-deep-dive-into-docker-overlay-networks-part-1/external-connectivity.png" alt="External Connectivity" width="600" style="margin: 0px auto;display:block;" />

### What about eth0, the interface connected to the overlay?
The interface peered with eth0 is not in the host network namespace. It must be
in another one. If we look again at the network namespaces:

```console
docker0:~$ sudo ip netns ls
e4b8ecb7ae7c
1-620dd59483
```

We can see a namespace called "1-c4305b67cd". Except for the "1-", the name of
this namespace is the beginning of the network id of our overlay network:

{% raw %}
    docker0:~$ docker network inspect demonet -f {{.Id}}
    620dd594834293e912bc17931d589c41bac318734d1084632f02da3177708bdc
{% endraw %}

This namespace is clearly related to our overlay network. We can look at the
interfaces present in that namespace:

```console
$ overns=1-620dd59483
$ sudo ip netns exec $overns ip -d link show
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default
    link/ether 3a:2d:44:c0:0e:aa brd ff:ff:ff:ff:ff:ff promiscuity 0
    bridge
5: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group default
    link/ether 4a:23:72:a3:fc:e3 brd ff:ff:ff:ff:ff:ff promiscuity 1
    vxlan id 256 srcport 10240 65535 dstport 4789 proxy l2miss l3miss ageing 300
    bridge_slave
7: veth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT group default
    link/ether 3a:2d:44:c0:0e:aa brd ff:ff:ff:ff:ff:ff promiscuity 1
    veth
    bridge_slave
```

The overlay network namespace contains three interfaces (and lo):
- br0: a bridge
- veth2: a veth interface which is the peer interface of eth0 in our container
  and which is connected to the bridge
- vxlan1: an interface of type "vxlan" which is also connected to the bridge

The vxlan interface is clearly where the "overlay magic" is happening and we are
going to look at it in details but let's update our diagram first:

<img src="/assets/2017-04-25-deep-dive-into-docker-overlay-networks-part-1/full-connectivity.png" alt="Full Connectivity" width="600" style="margin: 0px auto;display:block;" />

## Conclusion
This concludes part 1 of this article. In [part 2](http://techblog.d2-si.eu/2017/05/09/deep-dive-into-docker-overlay-networks-part-2.html), we will focus on VXLAN: what
is this protocol and how it is used by Docker.
