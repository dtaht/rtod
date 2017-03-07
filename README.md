# rtod: Routing Tables of Death

rtod allows for the rapid injection and removal of large numbers of
distinct routes per host, as a means to test the capabilities of
kernels, routing protocols, and routing daemons.

Do NOT run this on a production network. DO read this documentation.

The rtod script was developed to stress test kernel routing
features, daemons, and protocols as a means to measure their
behavior and make them better.

rtod is as robust as I can make it - running it in more extreme
modes tends to make tcp connections fail - so it has multiple means
to clean up after itself automatically. You will certainly see
network routing misbehave badly - and it might take 10s of
minutes for the network to recover, but it will, eventually. Usually.
Still, in addition to disrupting your routing until many minutes after the test ends:

- You may need to clean up nohup.out or kill off processes manually.

- You can easily melt a processor.

- You might end up with other system critical processes, like network manager or odhcpd, hung or spinning madly.

- You might run your kernel into the ground.

- You might run out of memory and engage the oom killer on stuff you don't want killed. 
 
- You might get your whole network into a state where you have to power cycle every router simultaneously.

- You might contribute to global warming.

Be *careful* with this tool.

# Setup

rtod uses the unallocated fc::/8 ULA address space to do its damage.

Make utterly sure your production network refuses to forward routes
injected by this script. You can easily insert a number of routes
far in excess of what most daemons, cpus, and protocols can handle.

Sample conf files are provided for babeld for setup of both the
routers under test and the routers that are not. I will add conf
files for bird, olsr, bmx, etc as time goes by.

Test on a small scale, first. Make absolutely sure nothing escapes.

# Uses: standalone

rtod's defaults are enough to mildly stress out most mesh networks.
They add 256 routes for 600 seconds. This generally does no damage.

These defaults are useful for reliably repeating routing behaviors
that are expected and normal, and observing metric evolution
elsewhere.

## rtod -r 1024

This is close to the figure being used in several production mesh
networks. This begins to stress out the cpu in mips routers.

## rtod -r 2048

This starts to stress out local route distribution mechanisms
over wifi multicast, and the protocol itself.

Low end ARM and MIPs CPUs start getting warm.

## rtod -r 4096

At this level the local network daemon tends to run out of cpu on 
just merging in the local kernel table. Other daemons interpreting
the protocol begin to struggle also.

Low end ARM and MIPs CPUs get very warm.

Routers using slow multicast drop off the network completely.

## rtod -r 10000

This tends to completely heat the local daemon with "interesting"
results on the rest of the network. Multicast becomes a huge 
bottleneck. Updates lag 10s of seconds, even minutes, behind.

Other daemons strain to keep up. Bad things happen.

Did I mention you should not run this on a production network?

In most of my tests thus far, anything above 2000 total routes begins
to degrade the connectivity of a network severely. With some tuning,
I've got babeld to about 5k routes but it is still barely hanging on
at that level,

## rtod -r 64000

Don't do this. 

# Uses: with other routers in the loop

Given that I have many routers, I push tests to them via pdsh. With
a smaller number of local route tables, they will generate
distinct routes based on their hostnames, and not cpu bottleneck
on getting routes out of the kernel.

````
pdsh -g chips rtod -r 128 # push out 512 routes total to my 4 "c.h.i.p"s.
````

This is a much better test of what actually happens with multiple routers
in play, in multiple places, and more complex scenarios can be easily
simulated. I have one with 32 simulated routers over a network diameter
of 12 hops and 3k routes, for example.

# Uses: Anycast emulation

by supplying the -H option to each invocation, we can have all routers 
announce they have routes to the same host, and confuse them appropriately

````
pdsh -g chips rtod -r 256 -H mytest
````

# Additional command line parameters

## -K - the Killme command.

This is used by rtod internally to make sure it
kills itself off and flushes its routes.

When used (often in desperation) at the command line, you can say:

````
rtod -K -t 0 
````

To have it flush all routes immediately.

## -r routes

Injects this number of routes, in the range 1-65534.

## -H hostname

Specify a different hostname to use. The hostname is used to
generate a md5 hash that is then turned into a usually distinct
ipv6 address. 

## -i IFACE

You can setup an alternative interface to use that is not a dummy.

If dummy interfaces are not available, and no iface is specified,
the ip6-localnet interface is used.

## -p PROTO

rtod uses a separate kernel protocol table to inject and manage
routes and addresses. The default is proto 50, which is unallocated
by the ietf and any other daemon I'm aware of. You can use any proto
you want, but, you should not - unless you want to confuse them -
inject routes into the kernel tables being managed by your daemons
in the first place.

This include static and boot routes as the rtod cleanup routine will wipe 
those out, too.

## -j jitter -J JITTER

Unimplemented, but the intent is to have routes disappear and re-appear
as if they were normally being lost and re-gained.

## -s Source specific offset

-s offset 

must be a power of two, that will inject source specific routes for a
subset of existing routes.

Unimplemented, presently.

## -t expires (in seconds) 

The duration of the test before routes are flushed.

## -m metric

Use a different metric to insert the routes than the default.

Unimplemented.

## -M Random Metric Range

Inject a random metric from 0 through -M when creating the
routes.

Unimplemented.

## -R route_table_dump_file

Inject an exact duplicate (rapidly) of a file saved with

````
ip route save proto 50 > route_table_dump_file
````

via

````
ip route restore proto 50 < route_table_dump_file
````

de-implemented, as it has a flaw of not inserting the expires figure,
and the whole point of rtod is to be able to survive abuse such
as this.

