# Pipework

**_Software-Defined Networking for Linux Containers_**

Pipework lets you connect together containers in arbitrarily complex scenarios. 
Pipework uses cgroups and namespace and works with "plain" LXC containers 
(created with `lxc-start`), and with the awesome [Docker](http://www.docker.io/).

##### Table of Contents
* [Things to note](#notes)  
  * Virtualbox
  * Docker
* [LAMP stack with a private network between the MySQL and Apache containers](#lamp)  
* [Docker integration](#docker_integration)  
* [Peeking inside the private network](#peeking_inside)  
* [Setting container internal interface](#setting_internal)  
* [Using a different netmask](#different_netmask)  
* [Setting a default gateway](#default_gateway)  
* [Setting routes on the internal interface](#route_internal) 
* [Connect a container to a local physical interface](#local_physical)  
* [Let the Docker host communicate over macvlan interfaces](#macvlan)  
* [Wait for the network to be ready](#wait_ready)  
* [Add the interface without an IP address](#no_ip)  
* [DHCP](#dhcp)  
* [Specify a custom MAC address](#custom_mac)  
* [Virtual LAN (VLAN)](#vlan)  
* [IPv6](#ipv6)
* [Secondary addresses](#secondary)
* [Traffic Control (QoS)](#traffic_control)
* [Support Open vSwitch](#openvswitch)  
* [Support Infiniband](#infiniband)
* [Cleanup](#cleanup)  
* [Debugging](#debug)
* [Experimental](#experimental)  


<a name="notes"/>
### Things to note

#### Virtualbox

**If you use VirtualBox**, you will have to update your VM network settings.
Open the settings panel for the VM, go the the "Network" tab, pull down the
"Advanced" settings. Here, the "Adapter Type" should be `pcnet` (the full
name is something like "PCnet-FAST III"), instead of the default `e1000`
(Intel PRO/1000). Also, "Promiscuous Mode" should be set to "Allow All".
If you don't do that, bridged containers won't work, because the virtual
NIC will filter out all packets with a different MAC address.  If you are
running VirtualBox in headless mode, the command line equivalent of the above
is `modifyvm --nicpromisc1 allow-all`.  If you are using Vagrant, you can add
the following to the config for the same effect:

```Ruby
config.vm.provider "virtualbox" do |v|
  v.customize ['modifyvm', :id, '--nicpromisc1', 'allow-all']
end
```

#### Docker

**Before using Pipework, please ask on the [docker-user mailing list](
https://groups.google.com/forum/#!forum/docker-user) if there is a "native"
way to achieve what you want to do *without* Pipework.**

In the long run, Docker will allow complex scenarios, and Pipework should
become obsolete.

If there is really no other way to plumb your containers together with
the current version of Docker, then okay, let's see how we can help you!

The following examples show what Pipework can do for you and your containers.


<a name="lamp"/>
### LAMP stack with a private network between the MySQL and Apache containers

Let's create two containers, running the web tier and the database tier:

    APACHE=$(docker run -d apache /usr/sbin/httpd -D FOREGROUND)
    MYSQL=$(docker run -d mysql /usr/sbin/mysqld_safe)

Now, bring superpowers to the web tier:

    pipework br1 $APACHE -a ip 192.168.1.1/24

This will:

- create a bridge named `br1` in the docker host;
- add an interface named `eth1` to the `$APACHE` container;
- assign IP address 192.168.1.1 to this interface,
- connect said interface to `br1`.

Now (drum roll), let's do this:

    pipework br1 $MYSQL -a ip 192.168.1.2/24

This will:

- not create a bridge named `br1`, since it already exists;
- add an interface named `eth1` to the `$MYSQL` container;
- assign IP address 192.168.1.2 to this interface,
- connect said interface to `br1`.

Now, both containers can ping each other on the 192.168.1.0/24 subnet.


<a name="docker_integration"/>
### Docker integration

Pipework can resolve Docker containers names. If the container ID that
you gave to Pipework cannot be found, Pipework will try to resolve it
with `docker inspect`. This makes it even simpler to use:

    docker run -name web1 -d apache
    pipework br1 web1 -a ip 192.168.12.23/24


<a name="peeking_inside"/>
### Peeking inside the private network

Want to connect to those containers using their private addresses? Easy:

    ip addr add 192.168.1.254/24 dev br1

Voilà!

<a name="setting_internal"/>
### Setting container internal interface ##
By default pipework creates a new interface `eth1` inside the container. In case you want to 
change this interface name like `eth2`, e.g., to have more than one interface set by pipework, use:

`pipework br1 web1 -i eth2 ...`

**Note:**: for infiniband IPoIB interfaces, the default interface name is `ib0` and not `eth1`.

<a name="different_netmask"/>
### Using a different netmask

The IP addresses given to `pipework` are directly passed to the `ip addr`
tool; so you can append a subnet size using traditional CIDR notation.

I.e.:

    pipework br1 $CONTAINERID -a ip 192.168.4.25/20

Don't forget that all containers should use the same subnet size;
pipework is not clever enough to use your specified subnet size for
the first container, and retain it to use it for the other containers.


<a name="default_gateway"/>
### Setting a default gateway

If you want *outbound* traffic (i.e. when the containers connects
to the outside world) to go through the interface managed by
Pipework, you need to change the default route of the container.

This can be useful in some usecases, like traffic shaping, or if
you want the container to use a specific outbound IP address.

This can be automated by Pipework, by adding the gateway address
after the IP address and subnet mask:

    pipework br1 $CONTAINERID -a ip 192.168.4.25/20@192.168.4.1


<a name="route_internal"/>
### Setting routes on the internal interface

If you add more than one internal interface, or perform specific use-cases, like multicast routing 
you may want to add other routes than the default one. 
This could be performed by adding network and masks after the gateway (comma-separated)

    pipework br1 $CONTAINERID -a ip 192.168.4.25/20@192.168.4.1 -r 192.168.5.0/25,192.168.6.0/24

Please note that the last added internal interface will take the default route


<a name="local_physical"/>
### Connect a container to a local physical interface

Let's pretend that you want to run two Hipache instances, listening on real
interfaces eth2 and eth3, using specific (public) IP addresses. Easy!

    pipework eth2 $(docker run -d hipache /usr/sbin/hipache) -a ip 50.19.169.157/24
    pipework eth3 $(docker run -d hipache /usr/sbin/hipache) -a ip 107.22.140.5/24

Note that this will use `macvlan` subinterfaces, so you can actually put
multiple containers on the same physical interface.


<a name="macvlan"/>
### Let the Docker host communicate over macvlan interfaces

If you use macvlan interfaces as shown in the previous paragraph, you
will notice that the host will not be able to reach the containers over
their macvlan interfaces. This is because traffic going in and out of
macvlan interfaces is segregated from the "root" interface.

If you want to enable that kind of communication, no problem: just
create a macvlan interface in your host, and move the IP address from
the "normal" interface to the macvlan interface. 

For instance, on a machine where `eth0` is the main interface, and has
address `10.1.1.123/24`, with gateway `10.1.1.254`, you would do this:

    ip addr del 10.1.1.123/24 dev eth0
    ip link add link eth0 dev eth0m type macvlan mode bridge
    ip link set eth0m up
    ip addr add 10.1.1.123/24 dev eth0m
    route add default gw 10.1.1.254

Then, you would start a container and assign it a macvlan interface
the usual way:

    CID=$(docker run -d ...)
    pipework eth0 $CID -a ip 10.1.1.234/24@10.1.1.254


<a name="wait_ready"/>
### Wait for the network to be ready

Since `docker create` allow to instantiate the container without starting it, 
there is no more reason for pipework to provide tooling to wait for the network. 

<a name="no_ip"/>
### Add the interface without an IP address

If for some reason you want to set the IP address from within the
container, you can use `0/0` as the IP address. The interface will
be created, connected to the network, and assigned to the container,
but without configuring an IP address:

    pipework br1 $CONTAINERID -a link


<a name="dhcp"/>
### DHCP

You can use DHCP to obtain the IP address of the new interface. Just
specify `dhcp` instead of an IP address; for instance:

    pipework eth1 $CONTAINERID -a dhcp

The value of $CONTAINERID will be provided to the DHCP client to use
as the hostname in the DHCP request. Depending on the configuration of
your network's DHCP server, this may enable other machines on the network
to access the container using the $CONTAINERID as a hostname; therefore,
specifying $CONTAINERID as a container name rather than a container id
may be more appropriate in this use-case.

You need three things for this to work correctly:

- obviously, a DHCP server (in the example above, a DHCP server should
  be listening on the network to which we are connected on `eth1`);
- a DHCP client (either `udhcpc`, `dhclient` or `dhcpcp`) must be installed
  on your Docker *host* (you don't have to install it in your containers,
  but it must be present on the host);
- the underlying network must support bridged frames.

The last item might be particularly relevant if you are trying to
bridge your containers with a WPA-protected WiFi network. I'm not 100%
sure about this, but I think that the WiFi access point will drop frames
originating from unknown MAC addresses; meaning that you have to go
through extra hoops if you want it to work properly.

It works fine on plain old wired Ethernet, though.


<a name="custom_mac"/>
### Specify a custom MAC address

If you need to specify the MAC address to be used (either by the `macvlan`
subinterface, or the `veth` interface), no problem. Just add it as the
command-line, as the last argument:

    pipework eth0 $(docker run -d haproxy) -a ip 192.168.1.2/24 -m 26:2e:71:98:60:8f

This can be useful if your network environment requires whitelisting
your hardware addresses (some hosting providers do that), or if you want
to obtain a specific address from your DHCP server. Also, some projects like
[Orchestrator](https://github.com/cvlc/orchestrator) rely on static
MAC-IPv6 bindings for DHCPv6:

    pipework br0 $(docker run -d zerorpcworker) -a dhcp -m fa:de:b0:99:52:1c

**Note:** if you generate your own MAC addresses, try remember those two
simple rules:

- the lowest bit of the first byte should be `0`, otherwise, you are
  defining a multicast address;
- the second lowest bit of the first byte should be `1`, otherwise,
  you are using a globally unique (OUI enforced) address.

In other words, if your MAC address is `?X:??:??:??:??:??`, `X` should
be `2`, `6`, `a`, or `e`. You can check [Wikipedia](
http://en.wikipedia.org/wiki/MAC_address) if you want even more details.

**Note:**  Setting the MAC address of an IPoIB interface is not supported.


<a name="vlan"/>
### Virtual LAN (VLAN)

If you want to attach the container to a specific VLAN, the VLAN ID can be
specified using the `[MAC]@VID` notation in the MAC address parameter.

**Note:** VLAN attachment is currently only supported for containers to be
attached to either an Open vSwitch bridge or a physical interface. Linux
bridges are currently not supported.

The following will attach container zerorpcworker to the Open vSwitch bridge
ovs0 and attach the container to VLAN ID 10.

    pipework ovsbr0 $(docker run -d zerorpcworker) -a dhcp -V @10

<a name="ipv6"/>
### IPv6

IPv6 global scope adressing is also supported, using the same options : 

	pipework eth0 eth0 $(docker run -d haproxy) -a ip 2001:db8::beef/64@2001:db8::1

**Note:** Docker 1.5 feature

<a name="secondary"/>
### Secondary addresses

You can attach secondary addresses the container, using the action `sec_ip` instead of `ip`

	pipework eth0 $(docker run -d haproxy) -a sec_ip 192.168.1.2/24
	pipework eth0 $(docker run -d haproxy) -a sec_ip 2001:db8::beef/64
	pipework eth0 $(docker run -d haproxy) -a sec_ip 2001:db8::face/64

<a name="traffic_control"/>
### Traffic Control (QoS)

You can play with traffic control on an internal container interface, to emulate network 
properties like bandwidth, packet drops, latency, protocol policing and marking, etc. 

Here, we provide a simple wrapper around tc, so you can keep the control on all parameters

    pipework eth0 $MYSQL -a tc qdisc add dev eth1 root netem loss 30%
    pipework eth0 $MYSQL -a tc qdisc add dev eth1 root netem delay 100ms

See `man tc` for more details

**Note:** as it is a wrapper, be sure that all you pipework arguments are befoire `-a tc ...` 

<a name="openvswitch"/>
### Support Open vSwitch

If you want to attach a container to the Open vSwitch bridge, no problem.

    ovs-vsctl list-br
    ovsbr0
    pipework ovsbr0 $(docker run -d mysql /usr/sbin/mysqld_safe) -a ip 192.168.1.2/24

If the ovs bridge doesn't exist, it will be automatically created

<a name="infiniband"/>
### Support Infiniband IPoIB

Passing an IPoIB interface to a container is supported.  However, the entire device
is moved into the network namespace of the container.  It therefore becomes hidden
from the host.  

To provide infiniband to multiple containers, use SR-IOV and pass
the virtual function devices to the containers.
  
<a name="cleanup"/>
### Cleanup

When a container is terminated (the last process of the net namespace exits),
the network interfaces are garbage collected. The interface in the container
is automatically destroyed, and the interface in the docker host (part of the
bridge) is then destroyed as well.

<a name="debug"/>
### Debugging

2 switchs makes you able to debug some tedious situations : 

-v logs every iproute2 calls

-x enable shell debugging (similar to sh -x pipework ...)

<a name="experimental"/>
### Experimental

TBD : test/kernel watch/...

- Tunnel interfaces (GRE/IPIP/IP6_TUNNEL) 

    pipework eth0 $(docker run -d haproxy) -i eth1 -a ipip 192.168.1.3
    pipework eth0 $(docker run -d haproxy) -a ipip 2001:db8::2 

If the container has more than one internal interface, specify the internal interface (-i) to attach 
the tunnel to the good device

No more driver/mode to remember (ipip, ip6_tunnel, ipip6, ip6ip6, gre, ip6_gre,...), pipeworks adapts itself 
to the right situation regarding your adressing scheme (doing ipv4-in-ipv4 or ipv6-in-ipv6 encapsulation)

Be careful about the MTU in these situations... (tunneling in the container over tunneling on the host may lead 
to problems).

- Clean OVS bridge unused ports

