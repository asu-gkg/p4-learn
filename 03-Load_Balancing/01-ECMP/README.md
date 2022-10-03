# Equal-Cost Multi-Path Routing

## Introduction

In this exercise, we will implement a layer-3 forwarding switch that can load balance traffic towards a destination across equal cost paths.
We will implement ECMP (Equal-Cost Multi-Path) routing to load balance traffic across multiple ports.
When a packet with multiple candidate paths arrives, our switch should assign the next-hop by hashing some fields from the
header and compute this hash value modulo the number of possible equal paths. For example, in the topology below, when `s1` has to send a packet to `h2`, the switch should determine the output port by computing: `hash(some-header-fields) mod 4`.
The switch does ECMP hashing on a per-flow basis to prevent out-of-order packets, which means that all packets with the same source and destination IP addresses and source and destination ports always hash to the same next hop.

<p align="center">
<img src="images/multi_hop_topo.png" title="Multi Hop Topology"/>
<p/>

To read more about ECMP, see the following [page](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-ecmp-flow-based-forwarding.html).

## Before Starting

As usual, we provide you with the following files:

- `p4app.json`: a configuration file that describes the topology we want to create with the help of mininet and p4-utils package.
- `network.py`: a Python script that initializes the topology using *Mininet* and *P4-Utils*. One can use `network.py` or `p4app.json` indifferently to start the network.
- `p4src/ecmp.p4`: p4 program skeleton to use as a starting point.
- `p4src/includes`: We split our p4 code into multiple files for the first time in today's exercise. In the `include` directory, you will find `headers.p4` and `parsers.p4` (you need to complete them as well).
- `send.py`: a small python script to generate packets with different TCP port numbers.

### Notes about p4app.json

We will use a new IP assignment strategy for this exercise (and the next one). If you have a look at `p4app.json` you will see that
the option is set to `mixed`. That means only hosts connected to the same switch will be assigned to the same subnet. Hosts connected
to a different switch will belong to a different `/24` subnet. If you use the namings `hZ` and `sXY` (e.g h1, h2, s1...), the IP assignment
goes as follows: `10.x.y.z`. Where `x` and `y` are the respective upper and the lower bytes of the gateway switch ID, and `z` is the host id. For example, in the topology above,
`h1` gets `10.0.1.1` and `h2` gets `10.0.6.2`.
 
You can find all the documentation about `p4app.json` in the `p4-utils` [documentation](https://nsg-ethz.github.io/p4-utils/usage.html#json). Also, you can find information about assignment strategies [here](https://nsg-ethz.github.io/p4-utils/usage.html#automated-assignment-strategies).

## Implementing the L3 forwarding switch + ECMP

To solve this exercise, we have to program our switch such that it can forward L3 packets when there is one possible next hop or more.
For that, we will use two tables: in the first table, we match the destination IP, and depending on whether ECMP has to be applied (for that destination), we set the output port or an `ecmp_group`.
Next, we apply a second table that maps (ecmp_group, hash_output) to egress ports.

This time you will have to fill the gaps in several files: `p4src/ecmp.p4`, `p4src/include/headers.p4` and `p4src/include/parsers.p4`.
Additionally, you will have to create a `cli` command file for each switch and name them `sX-commands.txt` (see inside the `p4app.json`).
We placed all the command files inside `sw-commands` directory to keep the workspace more organized.

To complete the exercise, you have to do the following:

1. Use the header definitions that we already provided.

2. Define the parser that can parse packets up to `tcp`. Note that we do not consider `udp` packets in this exercise for simplicity. This time you must define the parser in `p4src/include/parsers.p4`.

3. Define the deparser. Just emit all the headers in the correct order.

4. Define a match-action table that matches the IP destination address of every packet and has three actions: `set_nhop`, `ecmp_group`, `drop`.
Set the drop action as default.

5. Define the action `set_nhop`. This action takes two parameters: destination mac and egress port.  Use the parameters to set the destination mac and
`egress_spec`. Set the source mac as the previous destination mac (this is not what an actual L3 switch would do, but we do it for simplicity). A more realistic implementation would create a table that maps egress_ports to each switch interface mac address. However, since the source mac address is not very important for this exercise, do this swap for now). When sending packets from a switch to another switch, the destination MAC address is not very important; thus, you can use an arbitrary one. However, keep in mind that when the packet is sent to a host needs to have the correct destination MAC address.
Finally, decrease the packet's TTL by 1. **Note:** since we are in an L3 network when you send packets from `s1` to `s2` you have to use the dst mac of the switch interface, not the mac address of the receiving host, which is done in the very last hop.

6. Define the action `ecmp_group`. This action takes two parameters, the `ecmp_group_id` (14 bits) and the number of next hops (16 bits). This
action is one of the critical parts of the ECMP algorithm. You have to do several things:

   1. In this action, we will compute a hash function. To store the output, you need to define a metadata field. Define `ecmp_hash` (14 bits) inside
   the metadata struct in `headers.p4`. Use the [`hash`](https://github.com/p4lang/p4c/blob/57f54582a9401b8a89f8254738fca0f350dd557e/p4include/v1model.p4#L453) extern function to compute the hash of packets 5-tuple (src ip, dst ip, src port, dst port, protocol). The signature of a hash function is:
   `hash(output_field, (crc16 or crc32), (bit<1>)0, {fields to hash}, (bit<16>)modulo)`. You can find all the available hash functions [here](https://github.com/p4lang/p4c/blob/57f54582a9401b8a89f8254738fca0f350dd557e/p4include/v1model.p4#L403). For example, if you want to use `crc32`, your algorithm will be `HashAlgorithm.crc32`.
   2. Define another metadata field and call it `ecmp_group_id` (14 bits).
   3. Finally, copy the value of the second action parameter `ecmp_group` in the metadata field you just defined (`ecmp_group_id`). We use this as the key to match the second table.

**Note**: A lot of people asked why the `ecmp_group_id` is needed. In a few words, it is a level of indirection that allows you to map from one IP address to a set of ports, which does not have to be the four ports we use in this exercise. For example, you could have that for `IP(1)`, you use only the upper two ports, and for `IP(2)`, you load balance using the two lower ports.
Thus, by creating two ECMP groups, you can easily map any destination address to any set of ports.

7. Define the second match-action table used to set `ecmp_group` to the next hops. The table should have `exact` matches to the metadata fields
you defined in the previous step. Thus, it should match to the `meta.ecmp_group_id` and then to the output of the hash function `meta.ecmp_hash` (which will be
a value ranging from 0 to `NUM_NEXT_HOPS-1`). A match in this table should call the `set_nhop` action you already defined above, and a miss should mark the packet
to be dropped (set `drop` as default action).  This indirection enables us to use any subset of interfaces. For example imagine that in the topology above, we have `h2` and `h3` ( h3 does not exist, but just for the sake of the example), we could define two different `ecmp_group` (in the previous table), one that maps to port 2 and 4, and
one that maps to port 3 and 5. And then, in this table, we could add two rules per group to make the outputs `[0,1]` from the hash function match `[2,4]` and `[3,5]`, respectively.

8. Define the ingress control logic:

    1. Check if the ipv4 header was parsed (use `isValid`).
    2. Apply the first table.
    3. If the action `ecmp_group` was called during the first table-apply. Call the second table.
    Note: to know which action was called during an apply you can use a switch statement and `action_run`, to see more information about how to check which action was used, check out
    the [P4 16 specification](https://p4.org/p4-spec/docs/P4-16-v1.2.2.html#sec-invoke-mau)

9. In this exercise, we modify an IP header's field for the first time (remember we have to subtract 1 from the ip.ttl field). When doing so, the `ipv4` checksum field needs
to be updated; otherwise, other network devices (or receiving hosts) might drop the packet. For that purpose, the `v1model` provides an `extern` function that can be called
inside the `MyComputeChecksum` control to update checksum fields. In this exercise, you do not have to do anything. Instead, go to the `ecmp.p4` file and check how the code uses `update_checksum`.

10.  This time you have to write six `sX-commands.txt` files, one per switch. Note that only `s1` and `s6` need to have `ecmp_group` installed. For all
the other switches setting rules for the first table (using action `set_nhop`) will suffice. For `s1` you have to set a direct next hop towards `h1`, and an ecmp
group towards `h2`. Set the ecmp group with `id = 1` and `num_hops = 4`. Then define 4 rules that map from 0 to 3 to one of the 4 switch output ports using the second table.

## Testing your solution

Once you have the `ecmp.p4` program finished (and all the `commands.txt` files), you can test its behavior:

1. Start the topology (this will also compile and load the program).
   ```bash
   sudo p4run
   ```
   or
   ```bash
   sudo python network.py
   ```

2. Check that you can ping:

   ```bash
   mininet> pingall
   ```

3. Monitor the 4 links from `s1` that will be used during `ecmp` (from `s1-eth2` to `s1-eth5`). By doing this, you will be able to check which path each flow takes.

   ```
   sudo tcpdump -enn -i s1-ethX
   ```

4. Ping between two hosts:

   You should see traffic in only 1 or 2 interfaces (due to the return path)
   since all the ping packets of the same flow have the same 5-tuple.

5. Do [iperf](https://iperf.fr/iperf-doc.php) between two hosts:
   
   You should also see traffic in 1 or 2 interfaces (due to the return path).
   Since a flow's packets have the same 5-tuple, and thus the hash always returns the same index.

6. Get a terminal in `h1`. Use the `send.py` script.

   ```bash
   python send.py 10.0.6.2 1000
   ```

   This will send `tcp syn` packets with random ports. Now you should see packets going to all the interfaces since each packet will have a different hash.

### Some notes on debugging and troubleshooting

We have added a [small guideline](https://github.com/nsg-ethz/p4-learning/wiki/Debugging-and-Troubleshooting) in the documentation section. Use it as a reference when things do not work as
expected.
