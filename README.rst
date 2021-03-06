networking-vpp
==============

ML2 Mechanism driver and small control plane for the [VPP
forwarder](https://wiki.fd.io/view/VPP/What_is_VPP%3F)

This is a Neutron mechanism driver to bring the advantages of VPP to
OpenStack deployments.

It's been written to be as simple and readable as possible while
offering either full Neutron functionality or a simple roadmap to it.
While the driver is not perfect, we're aiming for

- robustness in the face of failures (of one of several Neutron
  servers, of agents, of the etcd nodes in the cluster gluing them
  together)
- simplicity
- testability - having failure cases covered is no good if you don't
  have a means to test the code that protects you from them

As a general rule, everything is implemented in the simplest way, for
three reasons: we get to see it working faster, we can test it, and
anyone that wants to join the project can make sense of it.

Your questions answered
-----------------------

### How do I use it?

There's a devstack plugin. You can add this plugin to your local.conf
and see it working. The devstack plugin now takes care of

- installing the networking-vpp code
- installing VPP itself (version 17.01)
- installing etcd
- using a QEMU version that supports vhostuser well

To get the best performance, this will use vhostuser sockets to talk to
VMs, which means you need a modern version of your OS (CentOS 7 and
Ubuntu 16.04 work). It also means that you need to run your VMs with a
special flavor that enables shared memory - basically, you need to set
up hugepages for your VMs, as that's the only supported way Nova does
this today. Because you're using pinned shared memory you are going to
find you can't overcommit memory on the target machine. The devstack
plugin converts all the default flavours to use shared memory.

If you want to build from components yourself, you can certainly get
this working with your own VPP build or with a newer QEMU version, but
you may want to read the files in devstack/ to work out how we choose to
configure the system.

We've made some effort to make this backward-compatible so that it will
work with older stable branches as well as the current master branch of
Neutron. You should find this will work with Newton, Mitaka and Liberty.

### How do I do devstack, then?

Before you start, add the following to your kernel options and reboot.

> iommu=pt intel\_iommu=on

You will need to set some configuration items in your local.conf to get
the system running.

One of the important things you need to know before you start is that
VPP is an entirely user-space forwarder. It doesn't live in the kernel,
it doesn't use kernel network devices; using DPDK, it steals NICs away
from the kernel for its own nefarious uses. So - to test VPP - you can
do one of two things:

- choose a device to give to VPP, remembering that it will not be
  useful for anything else (so it's a *really* good idea not to use
  the interface with the host IP address)
- use a loopback or TAP interface in VPP to keep it quiet (perfect for
  one node test systems).

I recommend the following bits of configuration and use them when I'm
testing. Make a 'local.conf' in your devstack directory that looks
something like this:

> [[local|localrc]]
> \# We are going to use memory in the system for 2M hugepages.  Pick
> \# a number you can afford to lose.  Here we're taking 2500 pages
> \# (about 5GB of memory) which works well in my 8GB test VM.
> NR\_HUGEPAGES=2500
>
> disable\_service q-agt \# we're not using OVS or LB
> enable\_plugin networking-vpp https://github.com/openstack/networking-vpp
> Q\_PLUGIN=ml2
> Q\_USE\_SECGROUP=True
> Q\_ML2\_PLUGIN\_MECHANISM\_DRIVERS=vpp
> Q\_ML2\_PLUGIN\_TYPE\_DRIVERS=vlan,flat
> Q\_ML2\_TENANT\_NETWORK\_TYPE=vlan
> ML2\_VLAN\_RANGES=physnet:100:200
> MECH\_VPP\_PHYSNETLIST=physnet:tap-0:1000:2000
>
> [[post-config|\$NOVA\_CONF]]
> [DEFAULT]
> \# VPP uses some memory internally.  reserved\_huge\_pages
> \# tells Nova that we cannot allocate the last 512 pages
> \# (1GB) of memory to VMs, because in practice it will be
> \# used already and those VMs won't start.  If you tweak
> \# the VPP options you can reduce this number, and 1GB is
> \# probably too much for a test VM, but it's the default
> \# for a 2 core machine.
> reserved_huge_pages=node:0,size:2048,count:64

and a 'startup.conf' file like this:

> unix {
>   nodaemon
>   log /tmp/vpp.log
>   full-coredump
>   startup-config /etc/vpp-startup.conf
> }
> 
> api-trace {
>   on
> }
> 
> dpdk {
>         socket-mem 128
> }
> 

There are a few settings up there you might want to tweak.

Firstly, it's important that you get the memory allocation right -
we're going to take the memory in your system, make some of it into
hugepages, and then hand those hugepages to VPP and OpenStack.

Above, the NR_HUGEPAGES setting says how many 2MB hugepages are
allocated from the system.  This is a balancing act - you need a
number that leaves normal memory behind for the OS and the OpenStack
processes, but VPP and the VMs you run will all come out of the
hugepage allocation.  2500 pages - about 5GB - works well on an
8GB system.

From that memory, VPP will use some.  the socket-mem line says how
much memory in MB it will use for each core.  The above line tells it
to give one core 128MB of memory (64 pages).  You can change this
number or make it a comma separated list to add memory to additional
cores, but again that's a good place to start.

VMs that run in VPP systems have to use hugepages for their memory, so
we have a little under 5GB of memory remaining in this example to give
to the VMs we run.

The 'reserved_huge_pages' is a count of hugepages that OpenStack will
not be allowed to give out to VMs - it works out there are 2500 pages
available, and this line tells it that 64 of those pages are not its
to give away (because VPP has used them).  If you get this line wrong,
you will end up with scheduling problems.

Secondly, you need to sort out an 'uplink' port.  This is the port on
your VM that is used to connect the OpenStack VMs to the world.  The
above startup.conf has the line:

> MECH\_VPP\_PHYSNETLIST=physnet:tap-0:1000:2000

That *tap-0* is the name of a VPP interface, and you can change it to
suit your setup.

VPP is designed specifically to take one whole interface from the
kernel and use it as the uplink.  If you have a DPDK compatible 1Gbit
card, the interface is typically *GigabitEthernet2/2/0* - but this
does depend a bit on your hardware setup, so you may need to run
devstack, then run the command 'sudo vppctl show int' - which will
list the interfaces that VPP found - fix the startup.conf file and try
again.  (If your situation is especially unusual, you willw need to go
look at VPP's documentation at <http://wiki.fd.io/> to work out how
VPP chooses its interfaces and things about how its passthrough
drivers work). If you're setting up a multinode system, bridge this
between the servers and it will form the Neutron dataplane link.

Another option is to use *loop0* - this is a loopback dewvice. Using
this. you can get things up and running, but you won't get access to
the tenant networks from outside of VPP (though you can still use the
'ip netns exec' trick through router namespaces). You can run two VMs
and talk between them by logging in on the console, for instance.

If you need a 'loop0' interface, you have to make VPP create it at startup.  Add the following line to your startup.conf file:

> unix {
> ...
>     startup-config /etc/vpp-commands.txt
> }

And create that /etc/vpp-commands.txt containing the line

> create loopback interface

A third option is half way between the other two.  You can use *tap-0*
in your configuration, and make a Linux kernel TAP device to connect
your host kernel to your VMs.  This means you can easily run a one
node setup without needing an extra NIC port, but you can still
connect to the networks inside OpenStack using that interface and any
VLAN subinterfaces you care to create.  You can even set up masquerade
rules so that your VMs can talk to the world though your machine's
kernel NIC.

To use a TAP device, set up the vpp-commands.txt file as above but put in the line

> tap connect uplink

When VPP runs, it will create a new TAP interface 'uplink', which you
can being up, address, bridge, etc. as you see fit.  That device is
bridged to the VLANs that the VMs are attached to.

After all this, run ./stack.sh to make devstack run.

### But VPP won't start!

You may need to add the kernel command line option 

> iommu=pt

to your kernel before VPP starts.  It depends on the Linux deployment
you're using.  Refer to the VPP documentation if you need more help.

### What overlays does it support?

Today, it supports VLANs and flat networks.

### How does it work?

networking-vpp provides the glue from the Neutron server process to a
set of agents that control, and the agents that turn Neutron's needs
into specific instructions to VPP.

The glue is implemented using a very carefully designed system using
etcd. The mechanism driver, within Neutron's API server process, works
out what the tenants are asking for and, using a special failure
tolerant journalling mechanism, feeds that 'desired' state into a highly
available consistent key-value store, etcd. If a server process is
reset, then the journal - in the Neutron database -contains all the
records that still need writing to etcd.

etcd itself can be set up to be redundant (by forming a 3-node quorum,
for instance, which tolerates a one node failure), which means that data
stored in it will not be lost even in the event of a problem.

The agents watch etcd, which means that they get told if any data they
are interested in is updated. They keep an eye out for any changes on
their host - so, for instance, ports being bound and unbound - and on
anything of related interest, like security groups. If any of these
things changes, the agent implements the desired state in VPP. If the
agent restarts, it reads the whole state and loads it into VPP.

### Can you walk me through port binding?

This mechanism driver doesn't do anything at all until Neutron needs to
drop traffic on a compute host, so the only thing it's really interested
in is ports. Making a network or a subnet doesn't do anything at all.

And it mainly interests itself in the process of binding: the bind calls
called by ML2 determine if it has work to do, and the port postcommit
calls push the data out to the agents once we're sure it's recorded in
the DB. (We do something similar with security group information.)

In our case, we add a write to a journal table in the database during
the same transaction that stores the state change from the API. That
means that, if the user asked for something, Neutron has agreed to do
it, and Neutron remembered to write all of the details down, it makes
it to the journal; and if Neutron didn't finish saving it, it
*doesn't* get recorded, either in Neutron's own records or in the
journal. In this way we keep etcd in step with the Neutron database -
both are updated, or neither is.

The postcommit calls are where we need to push the data out to the
agents - but the OpenStack user is still waiting for an answer, so
it's wise to be quick. In our case, we kick a background thread to
push the journal out, in strict order, to etcd. There's a little bit
of a lag (it's tiny, in practice) before etcd gets updated, but this
way if there are any issues within the cloud (a congested network, a
bad connection) we don't keep the user waiting and we also don't
forget what we agreed to do.

Once it's in etcd, the agents will spot the change and change their
state accordingly.

To ensure binding is done correctly, we send Nova a notification only
when the agent has definitely created the structures in VPP necessary
for the port to work, and only when the VM has attached to VPP. In this
way we know that even the very first packet from the VM will go where
it's meant to go - kind of important when that packet's usually asking
for an IP address.

Additionally, there are some helper calls to determine if this mechanism
driver, in conjunction with the other ones on the system, needs to do
anything. In some cases it may not be responsible for the port at all.

### How does it talk to VPP?

This uses the Python API module that comes with VPP (vpp\_papi). VPP has
an admin channel, implemented in shared memory, to exchange control
messages with whatever agent is running. The Python bindings are a very
thin layer between that shared memory system and a set of Python APIs.
We add our own internal layer of Python to turn vpp's low level
communcations into something a little easier to work with.

### What does it support?

For now, assume it moves packets to where they need to go. unless
they're firewalled, in which case it doesn't. It also integrates
properly with stock ML2 L3, DHCP and Metadata functionality.

### What have you just done?

The most interesting improvement since last time is security - this is
new with the ACL functionality added for VPP 17.01. This includes
security groups, the anti-spoof filters (including the holes for things
like DHCP), the allowed address pair extension and the port security
flag.

### Any known issues?

In general, check the bugs at
<https://bugs.launchpad.net/networking-vpp> - but worth noting:

-  Security groups don't yet support the remote\_security\_group\_id
   parameter. If you use this they will ignore it and accept traffic
   from any source.
-  Some failure cases (agent reset, VPP reset) leave the agent
   wondering what state VPP is currently in. For now, in these cases,
   we take the coward's way out and reset the agent and VPP
   simultaneously, recreating its state from what's in etcd. This
   works, and will not go wrong, but you'll see a pause in your VM
   traffic as it happens. At the moment, you'll most commonly see this
   on software upgrades. See below for what we're doing about this.

### What are you doing next?

We also keep our job list in <https://bugs.launchpad.net/networking-vpp>
anything starting 'RFE' is a 'request for enhancement'.

We are working towards more fault tolerance. Our aim is to be tolerant
of two other failure modes: the case where etcd is so busy that the
updates expire before agents receive them, and the case where the agent
restarts. In both of these cases, we want to *resync* -reconfigure VPP
just as much as necessary that it now has the right state, and ideally
without disrupting the traffic for VMs that are already attached and
whose ports are correctly configured. This is work soon to come; you'll
find the patch in the patch queue and you're welcome to pitch in and
help.

We will be implementing L3 in VPP. In this case, using the same etcd
and agent and an additiional Neutron L3 driver, you'll be able to use
VPP to create Neutron routers complete with NAT and floating
IPs. There's already an early version in the patch queue.

We will be implementing an overlay using LISP GPE, which has better
horizontal scale than VLAN based overlays.

There are more server threads running than absolutely necessary. This is
not likely to cause you any significant problems, but we'll be tuning up
the performance a bit in the near future.

We'll be dealing with a few of the minor details of a good Neutron
network driver, like sorting out MTU configuration.

At the moment, the agent runs as root. We want to lower its privilege to
improve security.

### What can I do to help?

At the least, just use it! The more you try things out, the more we find
out what we've done wrong and the better we can make it.

If you have more time on your hands, review any changes you find in our
gerrit backlog. All feedback is welcome.

And if you want to pitch in, please feel free to fix something - bug,
typo, devstack fix, massive new feature, we will take anything. Feel
free to ask for help in \#openstack-neutron or in the openstack-dev
mailing list if you'd like a hand. The bug list above is a good place to
start, and there are TODO comments in the code, along with a handful of,
er, 'deliberate' mistakes we put into the code to keep you interested
(*ahem*).

### Why didn't you use the ML2 agent framework for this driver?

Neutron's agent framework is based on communicating via RabbitMQ. This
can lead to issues of scale when there are more than a few compute hosts
involved, and RabbitMQ is not as robust as it could be, plus RabbitMQ is
trying to be a fully reliable messaging system - all of which work
against a robust and scalable SDN control system.

We didn't want to start down that path, so instead we've taken a
different approach, that of a 'desired state' database with change
listeners. etcd stores the data of how the network should be and the
agents try to achieve that (and also report their status back via etcd).
One nice feature of this is that anyone can check how well the system is
working - both sorts of update can be watched in real time with the
command

> etcdctl watch --recursive --forever /

The driver and agents should deal with disconnections across the board,
and the agents know that they must resync themselves with the desired
state when they completely lose track of what's happening.
