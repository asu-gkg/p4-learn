# Flowlet Switching

## Introduction

In the previous exercise, we implemented ECMP, a fundamental and widely used technique to load balance traffic across multiple equal cost paths. ECMP works very well when it has to load balance many small flows with similar sizes (since it
randomly maps them to one of the possible paths). However, real traffic does not look as described above. Real traffic is composed of many
small flows and few that are considerably bigger. This discrepancy makes ECMP suffer from a well-known performance problem, such as hash collisions,
in which few significant flows end up colliding in the same path. In this exercise, we will use the state and information provided by the simple_switch's
`standard_metadata` to mitigate the collision problem of ECMP by implementing flowlet switching on top.

Flowlet switching leverages the burstiness of TCP flows to achieve better load balancing. TCP flows tend to come in bursts (for instance, because
a flow needs to wait to get window space). Every time there is a gap that is big enough (i.e., 50ms) between packets from the same flow, flowlet switching
will rehash the flow to another path (by hashing an ID value together with the 5-tuple).

For more information about flowlet switching, check out this [paper](https://www.usenix.org/system/files/conference/nsdi17/nsdi17-vanini.pdf)

<p align="center">
<img src="images/multi_hop_topo.png" title="Multi Hop Topology"/>
<p/>


## Before Starting

As usual, we provide you with the following files:

- `p4app.json`: describes the topology we want to create with the help of mininet and p4-utils package.
- `network.py`: a Python scripts that initializes the topology using *Mininet* and *P4-Utils*. One can use indifferently `network.py` or `p4app.json` to start the network.
- `p4src/flowlet_switching.p4`: p4 program skeleton to use as a starting point.
- `p4src/includes`: In the includes directory you will find `headers.p4` and `parsers.p4` (which also have to be completed).
- `send.py`: a small python script to send bursts of packets that belong to the same flow.

## Implementing the flowlet switching enhancement

This exercise is an enhancement of ECMP, and thus you can start by copying all the code. You will use exactly the same headers,
parser, tables, and cli commands (so you do not need to write this part either).

To solve this exercise, you will have to use two registers, one for `flowlet_ids` (hash seed) and one to keep the last timestamp of
every flow. You must change the ingress logic slightly and define a new action to read/write the flowlet registers. And modify
the hash function used in ECMP, adding a new field (the `flowlet_id`) which will vary over time.

You will have to fill the gaps in several files: `p4src/flowlet_switching.p4`, `p4src/include/headers.p4`
and `p4src/include/parsers.p4`.

To complete this exercise, you have to do the following:

1. Like in the previous exercise, header definitions are already provided.

2. Define the parser that can parse packets up to `tcp`. Note that we do not consider `udp` packets for simplicity in this exercise. Define your parser in `p4src/include/parsers.p4`.

3. Define the deparser. Just emit all the headers.

4. Copy the tables and actions from the previous exercise. You will have to modify them slightly.

5. Define two registers `flowlet_to_id` and `flowlet_time_stamp` (for register sizing use the constant defined at the
beginning of `flowlet_switching.p4` file: REGISTER_SIZE, TIMESTAMP_WIDTH, ID_WIDTH). We will use these two registers to keep two things:

    1. In `flowlet_to_id` register we keep the id (a randomly generated number) of each flowlet, this id is now added to the hash function that devices the output port. As long as this id does not change, packets for that flow will stay on the same path.

    2. In `flowlet_time_stamp` register we keep the last timestamp for the last observed packet belonging to a flow.

6. Define an action to read the flowlet's register values (`read_flowlet_registers`). In this action, you will have to hash the 5-tuple
of every packet the index you will use to read the flowlet registers (to save the index, you will need to define a new metadata field with a width size of 16 bits). Using the index you got from the hash function, read flowlet id and last timestamp and save them in a metadata field (you
also have to define them). Finally, update the timestamp register with the latest packet's timestamp using `standard_metadata.ingress_global_timestamp`.

7. Define another action to update the flowlet id (`update_flowlet_id`). We will use this action to update flowlet ids when needed.
In this action, you have to generate a random number and then save it in the `flowlet_to_id` register (using the id you already computed previously).

8. Modify the `hash` function you defined in the ECMP exercise (`ecmp_group`); now, instead of just hashing the 5-tuple, you have to add the metadata field where you store the `flowlet_id` you read from the register (or you just updated).

9. Define the ingress control logic (keep the logic from the ECMP example and add):

    Before applying the `ipv4_lpm` table:

    1. Read the flowlet registers (calling the action)
    2. Compute the time difference between now and the last packet observed for the current flow.
    3. Check if the time difference is greater than `FLOWLET_TIMEOUT` (defined at the beginning of the file with a default
    value of 200ms).
    4. Update the flowlet id if the difference is greater. Updating the flowlet id will make the hash function output a new value.
    5. Apply `ipv4_lpm` and `ecmp_group` the same way you did in `ecmp`.

10. Copy the `sX-commands.txt` from the previous exercise.

## Testing your solution

Once you have the `flowlet_switching.p4` program finished you can test its behaviour:

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

3. Monitor the 4 links from `s1` that will be used during `ecmp` (from `s1-eth2` to `s1-eth5`). Doing this you will be able to check which path is each flow taking.

   ```bash
   sudo tcpdump -enn -i s1-ethX
   ```

4. Ping between two hosts:

   If you run a normal ping from the mininet cli, or using the terminal, by default it will send a ping packet every 1 second. In this
   case every ping should belong to a different flowlet, and thus it should be crossing different paths all the time.

5. Do iperf between two hosts:

   If you do iperf between `h1` and `h2` you should see all the packets cross the same interfaces almost
   all the time (unless you set the gap interval very small).


6. Get a terminal in `h1`. Use the `send.py` script.

   ```bash
   python send.py 10.0.6.2 1000 <sleep_time_between_packets>
   ```

   This will send `tcp syn` packets with the same 5-tuple. You can play with the sleep time (third parameter). If you set it bigger than your gap, packets should change
   paths, if you set it smaller (set it quite smaller since the software model is not very precise) you will see all the packets cross the same interfaces.

#### Some notes on debugging and troubleshooting

We have added a [small guideline](https://github.com/nsg-ethz/p4-learning/wiki/Debugging-and-Troubleshooting) in the documentation section. Use it as a reference when things do not work as
expected.



