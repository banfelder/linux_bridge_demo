# Linux Bridge Demonstration

This project demonstrates a linux box working as a Layer 2 network switch (or so-called `linux bridge`); the whole demonstration runs under Vagrant/VirtualBox.

To demonstrate a working switch (bridge), this project defines an environment with three virtual hosts running under VirtualBox:

1. h0: a bare host with IP address 192.168.60.10/24 running on eth1
2. h1: a bare host with IP address 192.168.60.11/24 running on eth1
3. b0: a host running linux bridge (i.e. acting as an L2 switch) that will allow h0 to ping h1. This switch has two network ports, eth1 and eth2. As it is just a switch, these ports don't have IP addresses assigned to them.

Note that all three hosts have an eth0 network interface. This is the NAT bridge that VirtualBox requires, and that you use to ssh from your VirtualBox host into each of the guest boxes. None of the eth0 interfaces are relevant to the demonstration of bridging.

A diagram of the modeled network:

          +----+
          |    |                                  +----+
     eth0 |    | eth1                        eth1 |    |
----------+ h0 +----------------------------------+    |
   NAT to |    | 192.168.60.10/24                 |    |
VBox host |    |                                  |    |
          +----+                                  |    | eth0
                                                  | b0 +----------
          +----+                                  |    | NAT to
          |    |                                  |    | VBox host
     eth0 |    | eth1                        eth2 |    |
----------+ h1 +----------------------------------+    |
   NAT to |    | 192.168.60.11/24                 |    |
VBox host |    |                                  +----+
          +----+

## Quick Start

`vagrant up`
`vagrant ssh b0`
`tcpdump -i eth1`

...and then in a second terminal...

`vagrant ssh h0`
`ping 192.168.60.11`

You should see successful pings of h1 from h0, and observe ARP and ICMP packets exchanged on the b0 switch.

## Implementation Notes

When defining the networking in Vagrant, we specify dummy IP addresses for b0:eth1 (192.168.60.9/24) and b0:eth2 (192.168.61.9/24) in the `mybox.vm.network` statements.
This causes Vagrant/VirtualBox to create two separate and distinct virtual networks for these interfaces.
If you look b0's VirtualBox network settings, you'll see that Adapter 2 is assigned to vboxnetX, and Adapter 3 is assigned to vboxnetY, where X != Y.

h0's eth1 is assigned a dummy IP address of 192.168.60.10/24 in the Vagrant/VirtualBox configuration.
Since this is in the same network address space as 192.168.60.9/24, Vagrant helpfully assigns h0:eth1 to the same vboxnetX as b0:eth1.

Similarly, h1's eth1 is assigned a dummy IP address of 192.168.61.11/24.
Since this is in the same network address space as 192.168.61.9/24, Vagrant helpfully assigned h1:eth1 to the same vboxnetY as b0:eth2.

A more complete network diagram is thus:

          +----+
          |    |                      +----------+                +----+
     eth0 |    | eth1                 |          |           eth1 |    |
----------+ h0 +----------------------+ vboxnetX +----------------+    |
   NAT to |    | 192.168.60.10/24     |          |                |    |
VBox host |    |                      +----------+                |    |
          +----+                                                  |    | eth0
                                                                  | b0 +----------
          +----+                                                  |    | NAT to
          |    |                      +----------+                |    | VBox host
     eth0 |    | eth1                 |          |           eth2 |    |
----------+ h1 +----------------------+ vboxnetY +----------------+    |
   NAT to |    | 192.168.60.11/24     |          |                |    |
VBox host |    |                      +----------+                +----+
          +----+

Note that all of the Vagrant driven `mybox.vm.network` statements specify the `auto_config: false` option, so none of these IP addresses are actually set up by Vagrant or VirtualBox.
The actually configuration of the network interfaces is done in the in-line provisioning scripts (`mybox.vm.provision` statements).
In these scripts we prefer iproute2 commands over the older utilities such as `ifconfig`; comments for h0's setup show equivalent commands.

It is important to tell Network Manager to ignore all ethN devices before configuring them explicitly; otherwise Network Manager tries to "help".

Linux bridging requires that the NICs that are part of the 'switch' be put in promiscuous mode. This means that they will accept inbound L2 frames with MAC address that are not their own.
To achieve this, we need to both tell VirtualBox to allow this (`vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]`), and then to actually put the NICs into promiscuous mode (`ip link set eth1 promisc on`).

b0:eth0 and b0:eth1 don't have their own IP addresses. This is ensured with the `ip address flush` commands.

The network configuration scripts used here are not idempotent, and do not persist configuration across reboots.
They are written for maximum clarity, and not intended for production use.

## Resources

Martin Brown's [http://linux-ip.net/html/index.html Guide to IP Layer Network Administration with Linux] was very helpful in preparing this demonstration project.
