---
author: Laurent Bernaille
layout: post
title: "Deep dive into Docker Overlay Networks: Part 3"
twitter: lbernail
keywords: docker overlay network,overlay network,vxlan,netlink,dockercon,linux networking
---

## Introduction
In [part1](/2017/04/25/deep-dive-into-docker-overlay-networks-part-1.html) of
this blog post we have seen how Docker creates a dedicated namespace for the
overlay and connect the containers to this namespace. 
In [part2](/2017/05/09/deep-dive-into-docker-overlay-networks-part-2.html) we
have looked in details at how Docker uses VXLAN to tunnel traffic between the
hosts in the overlay. In this third post, we will see how we can create our
own overlay with standard Linux commands.

## Manual overlay creation
First, if you have tried the commands from the two posts, we need to clean-up
our Docker hosts by removing all our containers and the overlay network:
```
docker0:~$ docker rm -f $(docker ps -aq)
docker0:~$ docker network rm demonet
docker1:~$ docker rm -f $(docker ps -aq)
```
The first we are going to do now is to create an network namespace called "overns":
```
sudo ip netns add overns
```
Now we are going to create a bridge in this namespace, give it an IP address and
bring the interface up:
```
sudo ip netns exec overns ip link add dev br0 type bridge
sudo ip netns exec overns ip addr add dev br0 192.168.0.1/24
sudo ip netns exec overns ip link set br0 up
```
The next step is to create a VXLAN interface and attach it to the bridge:
```
sudo ip link add dev vxlan1 type vxlan id 42 proxy learning dstport 4789
sudo ip link set vxlan1 netns overns
sudo ip netns exec overns ip link set vxlan1 master br0
sudo ip netns exec overns ip link set vxlan1 up
```
The most important command so far is the creation of the VXLAN interface. We
configured it to use VXLAN id 42 and to tunnel traffic on the standard VXLAN
port. We used two other options, proxy and learning, which we will explain
later when we will see what they are for. Notice that we did not create the
VXLAN interface inside the namespace but on the host and then moved it to the
namespace. This is necessary so the VXLAN interface can keep a link with our 
main host interface and send traffic over the network. If we had created the
interface inside the namespace (like we did for br0) we would not have been
able to send traffic outside the namespace.

Once we have run these commands on both docker0 and docker1, here is what we
have:
<img src="/assets/2017-06-01-deep-dive-into-docker-overlay-networks-part-3/overlay-1.png" alt="VXLAN interface and bridge in an overlay namespace">

Now we are going to create containers and connect them to our bridge. Let's
start with docker0. First, we create a container:
```
docker run -d --net=none --name=demo debian sleep 3600
```
We are going to need the network namespace for this container, we can find it
by inspecting the container (the second command is simply removing the path
information so we only get the id of the namespace).

{% raw %}
    ctn_ns_path=$(docker inspect --format="{{ .NetworkSettings.SandboxKey}}" demo)
    ctn_ns=${ctn_ns_path##*/}
{% endraw %}

Our container has no network connectivity because of the ```--net=none```
option. We now create a veth and move one of its endpoints (veth1) to our overlay
network namespace, attach it to the bridge and bring it up.
```
sudo ip link add dev veth1 mtu 1450 type veth peer name veth2 mtu 1450
sudo ip link set dev veth1 netns overns
sudo ip netns exec overns ip link set veth1 master br0
sudo ip netns exec overns ip link set veth1 up
```
The first command uses an MTU of 1450 which is necessary due to the overhead
added by the VXLAN header.

The last step is to configure veth2: send it to our container network namespace
and configure it with a MAC address (02:42:c0:a8:00:02) and an IP address
(192.168.0.2):
```
sudo ip link set dev veth2 netns $ctn_ns
sudo ip netns exec $ctn_ns ip link set dev veth2 name eth0 address 02:42:c0:a8:00:02
sudo ip netns exec $ctn_ns ip addr add dev eth0 192.168.0.2/24
sudo ip netns exec $ctn_ns ip link set dev eth0 up
```
We used the same addressing schem as Docker: the last 4 bytes of the MAC address
match the IP address of the container and the second one is the VXLAN id.

We have to do the same on docker1 with different MAC and IP addresses
(02:42:c0:a8:00:03 and 192.168.0.3). If you use the terraform stack from 
the github [repository](https://github.com/lbernail/dockercon2017), there is 
a helper shell script to attach the container to the overlay. We can use it on
docker1:
```
docker run -d --net=none --name=demo debian sleep 3600
./attach-ctn.sh demo 3
```
The first parameter is the name of the container to attach and the second one
the final digit of the MAC/IP addresses.

Here is the setup we have gotten to:
<img src="/assets/2017-06-01-deep-dive-into-docker-overlay-networks-part-3/overlay-2.png" alt="Connecting containers to our overlay">

Now that our containers are configured, we can test connectivity:
```
docker0:~$ docker exec -it demo ping 192.168.0.3
PING 192.168.0.3 (192.168.0.3): 56 data bytes
92 bytes from 192.168.0.2: Destination Host Unreachable
```
We are not able to ping yet. Let's try to understand why by looking at
the ARP entries in the container and in the overlay namespace:
```
docker0:~$ docker exec demo ip neighbor show
docker0:~$ sudo ip netns exec overns ip neighbor show
```
Both commands do not return any result: they do not know what is the MAC address
associated with IP 192.168.0.3. We can verify that our command is 
generating an ARP query by running tcpdump in the overlay namespace:
```
sudo ip netns exec overns tcpdump -i br0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
```
If we rerun the ping command from another window, here is the tcpdump output
we get:
```
17:15:27.074500 ARP, Request who-has 192.168.0.3 tell 192.168.0.2, length 28
17:15:28.071265 ARP, Request who-has 192.168.0.3 tell 192.168.0.2, length 28
```
The ARP query is broadcasted and received by our overlay namespace but does
not receive any answer. We have seen in [part 2](/2017/05/09/deep-dive-into-docker-overlay-networks-part-2.html)
that the Docker daemon populates the ARP and FDB tables and makes use of the
proxy option of the VXLAN interface to answer these queries. We configured
our interface with this option so we can do the same by simply populating
the ARP and FDB entries in the overlay namespace:
```
docker0:~$ sudo ip netns exec overns ip neighbor add 192.168.0.3 lladdr 02:42:c0:a8:00:03 dev vxlan1
docker0:~$ sudo ip netns exec overns bridge fdb add 02:42:c0:a8:00:03 dev vxlan1 self dst 10.0.0.11 vni 42 port 4789
```
The first command creates the ARP entry for 192.168.0.3 and the second one
configures the forwading table by telling it the MAC address is accessible
using the VXLAN interface, with VXLAN id 42 and on host 10.0.0.11.

Do we have connectivity?
```
docker0:~$ docker exec -it demo ping 192.168.0.3
PING 192.168.0.3 (192.168.0.3): 56 data bytes
^C--- 192.168.0.3 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```
No yet, which makes sense because we have not configured docker1: the ICMP
request is received by the container on docker1 but it does not know how
to answer. We can verify this on docker1:
```
docker1:~$ sudo ip netns exec overns ip neighbor show

docker1:~$ sudo ip netns exec overns bridge fdb show
0e:70:32:15:1d:01 dev vxlan1 vlan 0 master br0 permanent
02:42:c0:a8:00:03 dev veth1 vlan 0 master br0
ca:9c:c1:c7:16:f2 dev veth1 vlan 0 master br0 permanent
02:42:c0:a8:00:02 dev vxlan1 vlan 0 master br0
02:42:c0:a8:00:02 dev vxlan1 dst 10.0.0.10 self
33:33:00:00:00:01 dev veth1 self permanent
01:00:5e:00:00:01 dev veth1 self permanent
33:33:ff:c7:16:f2 dev veth1 self permanent
```
The first command shows, as expected, that we do not have any ARP information on 
192.168.0.3. The output of the second command is more surprising because
we can see the entry in the forwarding database for our container on docker0.
What happened is the following: when the ICMP request reached the interface,
the entry was "learned" and added to the database.
Let's add the MAC information on docker1 and verify that we can now ping:
```
docker1:~$ sudo ip netns exec overns ip neighbor add 192.168.0.2 lladdr 02:42:c0:a8:00:02 dev vxlan1

docker0:~$ docker exec -it demo ping 192.168.0.3
PING 192.168.0.3 (192.168.0.3): 56 data bytes
64 bytes from 192.168.0.3: icmp_seq=0 ttl=64 time=1.737 ms
64 bytes from 192.168.0.3: icmp_seq=1 ttl=64 time=0.494 ms
```

We have successfuly built an overlay with standard Linux commands:
<img src="/assets/2017-06-01-deep-dive-into-docker-overlay-networks-part-3/overlay-3.png" alt="Overview of our manual overlay">
