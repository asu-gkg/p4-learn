# Load Balancing with P4

In this week's exercise, you will implement two different layer-4 load balancers. You will start using a layer-3 forwarding table and ensure that traffic gets load balanced across equal cost paths.

You will implement an L3 switch with ECMP capabilities in [the first exercise](./01-ECMP).
In [the second exercise](./02-Flowlet_Switching), you will extend your code and make the load balancing more dynamic by changing the load balancing decision for different flowlets within the same TCP flow.

Finally, we offer [an optional exercise](./03-Dynamic_Routing) in which you will learn how to dynamically populate all the routing tables and load-balancing tables automatically using a P4 controller.
In this optional exercise, you will also learn how to read network topology information so that you can populate switch
tables for any topology!
