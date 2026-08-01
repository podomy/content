Reference: 10.1109/DSN.2002.1028914

> Several large scale peer to peer distributed process groups _running over
> the internet_ rely on a distributed membership maintenance sub-system.

> Examples of existing middleware systems that utilize a membership protocol include _reliable multicast_, and _epidemic style information dissemination_.

> These protocols find their applications in databases which need to reconcile recent disconnected updates. Or publish-subscribe systems (The message busses like kafka and all). Or large scale P2P systems.

They make an argument that the performance of the applications in **emerging** aspect depends critically on the reliability, scalability of the membership maintenance protocol within.

What the authors of the paper meant by emerging and the reason why they emphasized it is that in 2001-2002 production distributed systems were almost exclusively small-scale, tightly coupled clusters in a flat local network.

In these clusters the all-to-all heartbeating or monitoring had a time complexity of O(n^2), and back in time it was good enough.

The node count was not significant enough for the switches used to start dropping packets.

And the emerging applications in this case meant the early shift of 2000s.

- **P2P File Sharing & Networks**: BitTorrent, Napster, and Distributed Hash Tables (Chord, CAN, Pastry).
- **Multiplayer Online Games & Distributed Virtual Environments (DVEs)**: P2P match lobbies (e.g., early Xbox Live, DirectPlay) that had to keep track of active players over lossy home Internet connections, detect disconnects versus lag spikes, and execute seamless Host Migration when a player dropped out.
- **Global Data Infrastructure**: Multi-region database replication and distributed message busses.

Node count exploded from dozens to tens of thousands. All communicating over great distances which always resulted in packet loss.

And what the authors foresaw is that the membership subsystem with a big O complexity of n^2, O(n^2) or large O(n) gossip payloads, and specifically the failure detector itself saturates the network bandwidth before the actual application work even runs.

Let's say you had a packet loss happen on one of the nodes, where a healthy node got marked dead in the memberlist. It would trigger a check of all the systems again causing the traffic from these checks to saturate networks.

Unless an algorithm with O(1) time complexity of overhead was found the big networks would have no chance to exist, because the very heartbeats required to know who is alive would eat up the bandwidth you need.

Let us run some calculations for traditional heart beat protocols and SWIM protocol:

Assume an arbitrary 1000 node cluster.
Let N represent the number of nodes.

N = 1000

Traditionally it was in a way where every node must ping every other node once a second.

1000 \* 999 = 999,000 heart beat pings in total from all nodes

The network gets crashed by the health checks ALONE.

But now assume we had some packet loss or one of the nodes went down, now the N = 999. All of the other nodes immediately decide that one node went down.

Each one of them broadcasts "One node is dead!" to the other 998 nodes.

Those 998 nodes attempt to send a heartbeat to the node that went down at the same time, where we get a thundering herd. That's where the name comes from and why health probes (even today or in the future) should never ever be complex and should stay as simple as they can. The time complexity has to stay at O(1).

999 \* 998 = 997,002 notifications of each other before heartbeating the node that supposedly went down

And then finally another 998 calls heartbeating the node.

Now if the node that supposedly went down is actually alive, we ensure that 997,002 notifications and 998 heartbeats take it or the network down.

And if that failure causes another heartbeat failure in another node, the whole cluster just collapses. If decoupling between services is strong, then only one node and other nodes that would have the accidental packet loss will be going down.

[ insert a graph here of the scaling ]

Now consider the SWIM protocol, which has time complexity of O(1).

Each node pick only ONE random node to ping per protocol period.

1000 \* 1 = 1000

[ another graph here of the scaling of the two in different colors on the same chart ]

And on packet loss, instead of assuming the node went down. SWIM uses indirect probing and suspicion.

Node A does not assume Node B is dead. Node A asks 3 random nodes C, D, E: "Can you ping Node B for me?"

If one of them succeeds and gets an ACK from B, node C tells node A: "B is alive, your direct path just dropped a packet."

Node A updates its path metrics and moves on.

But then we have **suspicion**, if C, D, E also fail, Node A marks Node B as suspected, not dead. And it still keeps sending it heartbeats but one of them becomes a suspect signal.

Node b does **refutation** if it receives the suspect signal, node B goes: "Hey, I am, Alive!" and gossips or send an alive signal.
