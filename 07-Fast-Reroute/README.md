# Fast rerouting with Loop-Free Alternates (LFAs)

In this exercise, we will implement a mechanism to fast-reroute traffic upon a failure of an adjacent link towards a Loop-Free Alternate (LFA).
First, we'll introduce the problem and then give you an overview of the basic setup and your tasks.

<br>

## Introduction to fast rerouting and Loop-Free Alternates (LFAs)

**Topology:** Consider the following topology, composed of four routers and four hosts.

<img src="images/lfa_topo.png" width="400" alt="centered image" />

Each router is connected to all other routers and to one host.
The links between routers have different weights, as shown in the image.
The routers are connected to a central controller (not depicted) that computes the shortest paths between routers and fills up the forwarding tables. 

<img src="images/lfa_example.png" width="1200" alt="centered image" />

**Introduction to fast convergence:** 
- For the sake of example, consider the forwarding path towards `h4` (with destination prefix `10.4.4.2/24`) described in the leftmost figure above.
- With this forwarding configuration, consider the case in which the link between `S1` and `S2` fails. Whenever that occurs: (i) the adjacent routers `S1` and `S2` are able to detect this failure (almost) immediately, (ii) the central controller is notified about the failure, (iii) the central controller needs to re-compute the shortest paths, and (iv) the forwarding paths need to be updated in the data plane.
- During this time, all the traffic from `S2` and `S3` towards `10.4.4.2/24` is lost.
- While in this small demo network the impact is not significant, in large high-speed networks, recovering from a failure can take several *hundreds of milliseconds* and result in *many GB* of lost traffic.

As you have already learned in class, *fast-rerouting* techniques aim at closing this gap. The key insight is to 
install a (pre-computed) *backup* next hop, in addition to the "conventional" next hop, which we can use in case of failure. As such, whenever a router detects a local link failure, it can immediately forward traffic via the backup, preserving connectivity while the central controller re-computes the optimal forwarding paths with the new topology.

**Introduction to Loop-Free Alternates:**

As you have learned in class, the backup next-hops must be chosen with care to avoid forwarding loops.
In the presented example, if `S2` uses `S3` as a backup for traffic destined to `10.4.4.2/24`, it will generate a forwarding loop. Indeed, `S3` does not know about the link failure and will forward the traffic *back* to `S2` (c.f., middle figure).

This is where LFAs enter into the picture. Loop-Free Alternates (LFAs) are backup next hops that *do not result in loops*. 

For example: 
- In our case, `S4` is an LFA of `S2` for traffic towards `10.4.4.2/24`. Indeed, `S2` can start forwarding traffic to `S4`, and it will reach its destination. 
- Instead, `S3` is **not** an LFA of `S2` for traffic towards `10.4.4.2/24`. Indeed, `S3` would send the traffic towards `10.4.4.2/24` back to `S2`, generating a loop.

The condition of an LFA can be generalized to the following expression in terms of distances between nodes.
Let `D(X, Y)` be the distance between node `X` and `Y`. For router `S`, the next hop `N` is an LFA for destination `D`, if:

```
D(N, D) < D(N, S) + D(S, D)
```

:information_source: Not all topologies allow finding LFAs for any link and destination. In practice, networks are often *designed* such that this is possible.

Note that this condition considers primarily single link failures.
We will not deal with anything else in this exercise.
For additional consideration of node failures, links with shared risk, and more, please refer to [IP Fast Reroute RFC](https://tools.ietf.org/html/rfc5286).

<br>

## Exercise

The goal of the exercise is to enable switches to fast-reroute the traffic towards an LFA upon a failure of an adjacent link.

### Setup

We provide you with a basic setup. Concretely, you'll find the following files:

- `p4app.json` configures the topology introduced above with the help of mininet and the p4-utils package. Note that we disabled `pcap` logging to reduce disk usage. In case you want to use it, you will have to set the option to `true`.
- `network.py`: a Python script that initializes the topology using *Mininet* and *P4-Utils*. One can use indifferently `network.py` or `p4app.json` to start the network.
- `p4src/fast_reroute.p4`: the p4 program to use as a starting point. It already contains two register arrays: `primaryNH` allows looking up the port for a next hop, and `linkState` contains the local link information for a given port (you will be updating it): `0` if there are no issues, `1` if the link is down.
- `p4src/includes`: headers and parsers.
- `controller.py`: the central controller, already capable of installing forwarding rules.
- `heartbeat_generator.py`: a script to generate heartbeat messages from the control plane. The heartbeat messages enable the data planes to detect link failures.

### Architecture
We already provide you with the topology and a basic control- and data-plane implementation. At the high level, the architecture design is as follows: 

**Controller:** The controller is already capable of computing the shortest paths, even if some links have failed. 

**P4 Switches:** The switches are already capable of forwarding traffic to the (primary) next hop,
and contain a register array for the link states at each port. Your task will be to update these registers upon a link-failure detection. This will cause a fast reroute to the alternative path.

**Heartbeat protocol:** The controller sends a heartbeat packet for every port that is connected to another switch. According to the port that was set in the heartbeat packet the switch will send that packet over that port. The switch on the other side will receive that packet and remember its timestamp. Every time the switch receives a heartbeat message from the controller it will check the timestamp of the last hearbeat it got from the neighbor connected on the specified port. If it was too long ago it will assume the link to be down. So the heartbeats in this case are both used to check if the links are working by the heartbeat packets that get sent from switch to switch while at the same time the heartbeat packet received from the control plane makes sure the switch checks all its ports regularly. The p4 switch itself has no real concept of time and only does something when he receives a packet.

<p align="center">
    <img src="images/heartbeat.png" width="800" alt="centered image" />
</p>

### Your task
Your task is to extend the architecture design as follows: 

**Controller:** 

- For each switch and link, you must compute an LFA next hop to which the switch can fall back if the link fails.
- You need to install this LFA in the switches.
- While we are working with fixed topology, your controller should be able to work with other topologies as well. Do not hardcode LFAs into your code.

Hints: 
- On startup, the provided controller already provides full connectivity.
It configures the next hop indices per destination, and fills the register array for primary next hops.
This next hop index allows the switch to look up the relevant next hop ports.
In this exercise, we assign a unique next hop index to each host, and you do not need to change this. In practice, more efficient solutions are used, such as grouping destinations that take the same path through the network.

- You can put your full attention towards the `update_nexthops` method.
This method fills the register arrays with the actual next hops for each index.
For example, `h1` has the next hop id `0`. If there are no failures, the next
hop towards `h1` at switch `S2` is `S1`, located at port `2`.
Thus, the controller writes `2` to `primaryNH[0]` on `S2`.
If you fail the link between `S1` and `S2`, and `notify` the controller, it updates this register with `3`, the port towards `S3` (along with the registers in other switches).

- Your task is to extend this function to not only install the primary next hop,
but also a backup next hop.
You will need to coordinate this with your p4 code, that is, you will first need to update your code such that the controller can actually store the backup next hop somewhere.

When computing the backups, keep in mind that the backup next hops depend on the source router, primary next hop, and the destination.

:information_source: The host itself is also a next hop, although you do not need to compute a backup next hop here, as there is only one link available.

- Finally, you need to make sure that you do not install just *any* backup, but an LFA.
To check the LFA condition, you likely need the distances between nodes.
The method `dijkstra` provides you with both the (shortest) distances and paths for each pair of nodes in the network:

```python
failures = (given as input)
distances, paths = self.dijkstra(failures=failures)

distances['s1']['h3']  # Distance from s1 to h3.
paths['s1']['h3']      # Path from s1 to h3.
```

:information_source: Every time you call `dijsktra`, the shortest paths are recomputed, so make sure to not call it unnecessarily often, and re-use its output.

**Data plane:** 

- Detect the heartbeat messages sent from the controller. The heartbeat header includes three fields. The `from_cp` field indicates the heartbeat is received from the controller, and the `port` field indicates for which port this message is intended.
- For the indicated port, get the corresponding timestamp value. This value shows the latest observed heartbeat that has arrived from that port.
- If the time difference between the controller heartbeat and the latest heartbeat received from the adjacent switch is above a threshold, we can infer the link is down. Clone the heartbeat back to the controller to notify the link failure. Send the original heartbeat to its intended destination for other switches to receive.
- If the heartbeat arrives from neighbor switches, update the timestamp value for the received port.
- If the `linkState` register indicates that a link is down, use the backup link instead for forwarding traffic.

:information_source: The controller needs to populate the different primary and backup next hops *prior* to the failure.

Hints: 
- The p4 provided program first applies the table `ipv4_lpm`, which matches on the destination prefix using longest-prefix matching (`lpm`).
However, this table does not immediately map to an egress port, but rather to a next hop index.
These indices are installed once when the controller starts, and need not be modified again.

- Using this index, the switch can lookup the corresponding next hop egress port in the `primaryNH` register array.
The controller initially populates these registers, and updates them after failures.

- In addition to the primary next hop, you need to implement a way to look up a backup next hop (your LFA).
It might be useful to implement another register array similar to `primaryNH`, but other solutions are also possible.

- Finally, you need to put everything together and choose the primary if its link is up, and the backup otherwise.
You can find this information in the `linkState` register array.
The link state of port `X` is stored at index `X` in the register array.
It is `0` if there are no errors, and `1` if the link has failed.
Keep in mind that you *first* need to look up the port of the primary, before you can check whether the link at this port is up.

#### Note on IP addresses

For this exercise, we use the IP assignment strategy `l3`, which places each host in a different network.
The IP assigned to host `hX` connected to the switch `SY`
is as follows: `10.Y.X.2`. For example, in the topology above,
`h1` gets `10.1.1.2/24` and `h2` gets `10.2.2.2/24`.
You can find all the documentation about `p4app.json` in the `p4-utils` [documentation](https://nsg-ethz.github.io/p4-utils/usage.html#json).

<br>

## Testing your solution

### Startup

1.  Start the topology by executing `sudo p4run` (or `sudo python network.py`). This starts mininet and all the p4 switches. When mininet is running, open a second terminal or tab to execute the controller. Execute `sudo python controller.py`. This starts the controller script, which computes all shortest paths in the network and installs the corresponding forwarding rules. Afterwards, the controller starts sending heartbeat messages and listening to the switches for failure notifications.
The warnings due to unused variables can be safely ignored in this step. You can see the traffic/heartbeats between the CPU and the switches using `sudo tcpdump -en -i sX-cpu-eth1` where `X` is the switch number.

    ```bash
    sudo python controller.py --heartbeat_frequency 0.5 --notification_delay 10
    ```

2. The controller has two
   parameters. First you can set the `heartbeat_frequency` in seconds. And
   secondly, you can set the `notification_delay`. As expalined above, once the
   dataplane detects the failure, it instantaneously reports it to the control
   plane which will update the forwarding state as well as the backups.

    Make sure your `heartbeat_frequency` is smaller than the constant threshold
    defined in the P4 program (`#define THRESHOLD 48w1000000`) otherwise you
    might have false positives. By default, you will find the heart beat set to
    1 second. Thus, it will take at least 1 second before the failure can be
    detected by the dataplane. Feel free to play with the values.


3.  Verify that you can ping:
    ```bash
    mininet> pingall
    ````


### Failing links

- After running the network and the controller, you should be able to ping between hosts, or even run a `pingall` since the provided controller code already populates the basic forwarding rules. You can now fail a specific link in the mininet CLI. For example, executing `link s1 s2 down` command in the mininet CLI fails the link between `S1` and `S2`.
After executing this command, try to `pingall` from the mininet CLI. You will see that the routes using this link are now unreachable.

- The routes can become reachable again if the data plane detects the failure and falls back to the backup next hop. The data plane updates the `linkState` registers to keep track of down ports.

- The failure detection in the data plane results in a notification packet destined to the controller. The notification triggers the `failure_notification` method in `controller.py`, which updates all forwarding tables. To simulate the delay between a failure detection and the controller update, we have a `notification_delay` parameter in the controller script. This artificial delay enables you to observe the transition period between a failure detection and the controller update.

### Test example

Let us consider a failure of the link between `S1` and `S2` and will focus on the rerouting in `S2` for the traffic going to `10.4.4.2/24`, similar to the introduction above.

4.  Let us run the example in the figure above: we will monitor five links: S1-h1, S4-h4, and the three adjacent links of `S2`.

    To visualize these five links altogether, we could open separate tcpdumps, or we can use `speedometer`.

    First you need to install `speedometer` with:
    ```bash
    sudo apt-get install speedometer
    ```

    Since `speedometer` is not Python3 compliant, we have to force its execution with Python2. This can be done by running the following command.
    ```bash
    sudo sed -i '1s+^.*$+#!/usr/bin/env python2+' $(which speedometer)
    ```

    Then you can run the following command.
    ```bash
    speedometer -t s2-eth1 -t s2-eth2 -t s2-eth3 -t s2-eth4 -t s4-eth1
    ```

    :information_source: To see the interface names for all switches you can write `net` in the mininet CLI:
    ```
    mininet> net
    h1 h1-eth0:s1-eth1
    h2 h2-eth0:s2-eth1
    h3 h3-eth0:s3-eth1
    h4 h4-eth0:s4-eth1
    s1 lo:  s1-eth1:h1-eth0 s1-eth2:s2-eth2 s1-eth3:s4-eth3 s1-eth4:s3-eth4
    s2 lo:  s2-eth1:h2-eth0 s2-eth2:s1-eth2 s2-eth3:s3-eth2 s2-eth4:s4-eth4
    s3 lo:  s3-eth1:h3-eth0 s3-eth2:s2-eth3 s3-eth3:s4-eth2 s3-eth4:s1-eth4
    s4 lo:  s4-eth1:h4-eth0 s4-eth2:s3-eth3 s4-eth3:s1-eth3 s4-eth4:s2-eth4
    ```

5.  Start a high frequency ping from h2 to h4. Do not use the `mininet` cli
    since we will need it to introduce the link failures. If you are running the
    speedometer, you will see traffic in h2 and h4 direct interfaces, as well as
    in `s2-eth2`.
    ```
    > mx h2
    > ping 10.4.4.2 -i 0.001
    ```

6.  Fail the link S1-S2. For that you can use the mininet cli:
    ```
    mininet> link s1 s2 down
    ```

7. The detection should automatically happen in the dataplane. After 1 second,
   switches will enable the alternative paths. Then, after `notification_delay`
   the controller will update all the switch state with the new optimal paths
   and alternates.

8. We will first try with a threshold of 1 second in the P4 switch, a heart beat
   frequency of 500ms, and a delayed notification of 10 seconds. We set the
   delay quite high so you can see how the switch uses the alternate path and
   once the controller gets notified, the traffic gets sent to an optimal path.

   Once you run your solution you should see something like the image below.
   In the image, can see that `S2` quickly reroutes the traffic to `S4`, which is the LFA
   (`s2-eth4`). After the controller gets notified (10 seconds after) it
   recomputes the shortest paths, `S2` forwards to traffic to `S3`, the new
   primary next hop (traffic crosses `s2-eth3`).

<p align="center">
    <img src="images/speedometer_1.png" width="400" alt="centered image" />
</p>


9. Now, let us try with some smaller parameters. Modify the P4 program and set
   the detection threshold to 200ms (`#define THRESHOLD 48w200000`). Now
   increase the heartbeat frequency and set notification delay to 0. For
   example:

   ```
   sudo python controller.py --heartbeat_frequency 0.1 --notification_delay 0
   ```

   Now, your speedometer output should look like the image below:

<p align="center">
    <img src="images/speedometer_2.png" width="400" alt="centered image" />
</p>

   You can see that by default, now, there is traffic in all the interfaces,
   even if you do not ping. That is because we increased the heartbeat frequency
   which leads to some traffic overhead (not even visible). The biggest
   difference from the previous scenario, is that upon failure detection the
   controller gets instantaneously notified. As consequence, we can see the LFA
   is practically not used at all. Traffic moves from `s2-s1` to `s2-s3`
   instantaneously. Note that, there is a very small spike in traffic in
   `s2-s4`.