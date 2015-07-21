macvlan-docker-plugin
=================

### Pre-Requisites

1. Install the Docker experimental binary from the instructions at: [Docker Experimental](https://github.com/docker/docker/tree/master/experimental). (stop other docker instances)
	- Quick Experimental Install: `wget -qO- https://experimental.docker.com/ | sh`

### QuickStart Instructions


1. Start Docker with the following. (TODO: since macvlan doesnt use a bridge, should we point to the parent link name?)
`docker -d --default-network=macvlan:docker0`

2. Next start the plugin. A pre-compiled x86_64 binary can be downloaded from the [binaries](https://github.com/gopher-net/macvlan-docker-plugin/binaries) directory.

	```
	$ wget -O ./macvlan-docker-plugin https://github.com/gopher-net/macvlan-docker-plugin/binaries/macvlan-docker-plugin-0.1-Linux-x86_64
	$ chmod +x macvlan-docker-plugin
	$ ./macvlan-docker-plugin
	```

3. Start the plugin with `./macvlan-docker-plugin` for debugging or just extra logs from the sausage factory, add the debug flag `./macvlan-docker-plugin -d`

4. Run some containers and verify they can ping one another with `docker run -it --rm busybox` or `docker run -it --rm ubuntu` etc, any other docker images you prefer. Alternatively,`docker run -itd busybox`

    * Or use the script `release-the-whales.sh` in the `scripts/` directory to launch a bunch of lightweight busybox instances to see how well the plugin scales for you. It can also serve as temporary integration testing until we get CI setup. See the comments in the script for usage. Keep in mind, the subnet defined in `cli.go` is the temporarily hardcoded network address `192.168.1.0/24` and will hand out addresses starting at `192.168.1.2`. This is very temporary until we bind CLI options to the driver data struct.

5. The default macvlan mode is `bridge`. This works much like traditional vlans where a ToR switch or some other router would be the gateway in a data center. Currently, in order to have two subnets talk to one another it requires an L3 router. This is typical in the vast majority of DCs today. Again, this is just early, more to come and tons of ways to help if interested.

 **Additional Notes**:
 - The argument passed to `--default-network` the plugin is identified via `macvlan`. More specifically, the socket file that currently defaults to `/usr/share/docker/plugins/macvlan.sock`. If the filepath does not exist at runtime it will be created.
 - The networks are temporarily hardcoded. That and more will be configurable via flags. (Help us define and code those flags). Flags still need to be bound to structs also.
 - The containers are brought up on a flat bridge. This means there is no NATing occurring. A layer 2 adjacency such as a VLAN or overlay tunnel is required for multi-host communications. If the traffic needs to be routed an external process to act as a gateway (on the TODO list so dig in if interested in multi-host or overlays).
 - Download a quick macvlan video recording [here](https://www.dropbox.com/s/w0gts0kjs580k78/Macvlan-demo.mp4?dl=1).
 - Since this plugin uses netlink, a Linux host that can build [vishvananda/netlink](https://github.com/vishvananda/netlink) library is required.

### Example Output (This section is temporary until network config is user configurable):

Since the network parameters are hardcoded for the next day or two, I think it makes sense to show the output. This will be removed as soon as the network parameters are user defined.

**Note:** All of the following values can be modified in `cli.go` and will be configurable via plugin parameters/config over the next couple of nights of hacking and this section will be removed at that point.

 - The network address is `192.168.1.0/24` and the gateway is `192.168.1.1`. The gateway address is an external L3 router (home router, DC L3 leaf node, iptables on a Linux host etc.).
 - The hardcoded interface on the docker host is `eth1` (underlying OS that Docker is running on).

    ```
    ip addr show eth1
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
         link/ether 00:0c:29:c7:a0:3c brd ff:ff:ff:ff:ff:ff
         inet 192.168.1.254/24 brd 192.168.1.255 scope global eth1
    ```

 - Start Docker `docker -d -D --default-network=macvlan:foo`
 - Start macvlan-plugin-docker either using `go run main.go` or run the included binary `/macvlan-docker-plugin` in the binaries directory.


 - Container output:

    ```
    docker run -i -t --rm busybox
    # ifconfig eth0
    eth0      Link encap:Ethernet  HWaddr 72:3A:D8:99:30:A8
           inet addr:192.168.1.2  Bcast:0.0.0.0  Mask:255.255.255.255
           UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
           RX packets:1 errors:0 dropped:0 overruns:0 frame:0
           TX packets:7 errors:0 dropped:0 overruns:0 carrier:0
           collisions:0 txqueuelen:0
           RX bytes:60 (60.0 B)  TX bytes:578 (578.0 B)

    # ip route
    default via 192.168.1.1 dev eth0
    192.168.1.0/24 dev eth0  scope link

    / # ping 8.8.8.8
    PING 8.8.8.8 (8.8.8.8): 56 data bytes
    64 bytes from 8.8.8.8: seq=0 ttl=53 time=23.957 ms
    ^C
    --- 8.8.8.8 ping statistics ---
    1 packets transmitted, 1 packets received, 0% packet loss
    round-trip min/avg/max = 23.957/23.957/23.957 ms
    exit
    ```

 - The driver output will look something like the following as a container is created and then removed:

    ```
    go run main.go
    INFO[0000] Macvlan network driver initialized successfully
    INFO[0002] The allocated container IP is: [ 192.168.1.2 ]
    INFO[0002] Created Macvlan port: [ c78c9 ] using the mode: [ bridge ]
    INFO[0902] Removing unused macvlan link [ c78c9 ] from the removed container
    ```

### Hacking and Contributing

Yes!! This is a purely community project by folks hacking in their free time to provide references for plugging in to Docker via libnetwork remote APIs along with simple deployments for the array of common network architectures deployed in existing data centers today. Please see issues for todos or add todos you would like to add or that we can help you get accomplished if you are new to Docker, Go, networks or programming. This is an excting place and time to evolve with the community. The only rule here is no jerks :)

