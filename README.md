Non-Exclusive Leader Induction in Highly Available Systems
===

Published in [https://github.com/obsidiandynamics/NELI](https://github.com/obsidiandynamics/NELI) under a BSD (3-clause) license.


# Abstract
The Non-Exclusive Leader Induction (NELI) protocol provides a mapping from a set of work roles to one or more assignees responsible for fulfilling each role. Its utility is in environments where it’s imperative to maintain near-continuous system availability at the expense of occasional work duplication, and where the duplication of work, while generally undesirable, does not lead to an incorrect outcome. NELI is a simple unimodal protocol that is relatively straightforward to implement, building on a shared ledger service that’s capable of atomic partition assignment, such as Kafka or Kinesis. The latter makes NELI particularly attractive for use in Cloud-based computing environments. 

It can also be shown that with trivial modifications, NELI can be used in exclusive mode, trading availability for leader singularity, and multiple disjoint NELI groups may be used to satisfy strong continuous availability in non-exclusive mode.


# Overview
This text describes the Non-Exclusive Leader Induction (NELI) protocol built on top of a shared, partitioned ledger capable of atomic partition balancing (such as Apache Kafka or Amazon Kinesis). The protocol yields a non-exclusive leader in a group of contending processes for one of a number of notional roles, such that each role has at least one leader assigned to it. The number of roles is dynamic, as is the number of contending processes. This protocol is useful in scenarios where &mdash;

* There are a number of roles that need fulfilling, and it's desirable to share this load among several processes (likely deployed on different hosts);
* While it's undesirable that a role is simultaneously filled by two processes at any point in time, the system will continue to function correctly. In other words, this may create duplication of work but cannot cause harm (the _safety_ property);
* Availability of the system is imperative; it's required that at least one leader is assigned to a role at all times so that the system as a whole remains operational and progress is made on every role (the _liveness_ property);
* The number of processes and roles is fully dynamic, and may vary on an _ad hoc_ basis. Processes and roles may be added and removed without reconfiguration of the group or downtime. This accommodates rolling deployments, auto-scaling groups, and so forth;
* The use of a dedicated Group Membership Service (GMS) is, for whatever reason, deemed unviable, and where an alternate primitive is sought. Perhaps the system is deployed in an environment where a robust GMS is not natively available, but other capabilities that may internally utilise a GMS may exist. Kinesis in AWS is one such example.

**Note**: The term _induction_ is used in favour of _election_ to convey the notion of partial autonomy in the decision-making process. Under NELI, leaders aren't chosen directly, but induced through other phenomena that are observable by the affected group members, allowing them to infer that new leadership is in force. The protocol is also eventually consistent, in that although members of the group may possess different views at discrete moments in time, these views will invariably converge. This is contrary to a conventional GMS, where leadership election is the direct responsibility of the GMS, and is directly imparted upon the affected parties through view changes or replication, akin to the mechanisms employed in Zab [3] and Raft [4], and their likes.


# Practical implications
## Work assignment
Consider a system for dissemination of financial pricing data to multiple subscribers. If this is a time-critical service, then it's imperative that prices are always provided in a timely manner and the downtime of any price provider is to be avoided. A traditional high availability (HA) setup with fail-over provides continuity of price data; however, gaps during the fail-over process will be observed. A simple solution is to introduce active-active redundancy; however, this leads to duplication of pricing information. 

In the above scenario, a non-exclusive leader solves the availability problem without introducing excessive data duplication. (In fact, duplication may only be observable during a leadership transition.) Furthermore, non-exclusive leadership may be used as a load-balancing technique, such that one leader is chosen for each of the independent financial instruments for which a stream of pricing data exists. So a distributed cluster of processes may share the load of publishing price streams without overlapping under normal conditions; no minority subset of processes is responsible for publishing of the majority of data streams.

## Consistent hashing
In a dynamically sizeable, distributed cluster where some deterministic function is used to map a key to a shard, it is often highly desirable to minimise the amount of reassignment during the growth and contraction of the cluster. By mapping a fixed number of notional _hash slots_ to roles, then subsequently using NELI to arbitrate role assignment for a dynamically variable set of processes _P_ (deployed on hosts across the cluster), we achieve a stable mapping from processes to hash slots that varies minimally with membership changes in _P_. Taking the hash of a given key, modulo the number of hash slots, we arrive at the process that is singly or jointly responsible for managing the appropriate shard.


# Protocol definition
A centrally-arbitrated topic _C_ is established with set of partitions _M_. (Assuming _C_ is realised by a broker capable of atomic partition assignment, such as Kafka or Kinesis.) A set of discrete roles _R_ is available for assignment, and a set of processes _P_ contend for assignment of roles in _R_. The number of elements in the set _M_ may vary from that of _R_ which, in turn, may vary from _P_.

Each process in _P_ continually publishes a message on all partitions in _M_ (each successive message is broadcast a few seconds apart). The message has no key or value; the producing process explicitly specifies the partition number for each published message. As each process in _P_ publishes a message to _M_, then each partition in the set _M_ is continually subjected to messages from each process. Corollary to this, for as long as at least one process in _P_ remains operational, there will be at least one message continually published in each partition in _M_. Crucial to the protocol is that no partition may 'dry up'.

Each process _p_ in _P_ subscribes to _C_ within a common, predefined consumer group. As per the broker's partition assignment rules, a partition will be assigned to at most one consumer. Multiple partitions may be assigned to a single consumer, and this number may vary slightly from consumer to consumer. Note &mdash; this is a fundamental assumption of NELI, requiring a broker that is capable of arbitrating partition assignments. This dynamic set _P_ fits the broad definition of dynamic membership as described in Birman [1]. The term _NELI group_ is used to refer to a set _P_ operating under a distinct consumer group. (An alternate consumer group for _P_ implies a completely different NELI group, as partition assignment within the broker is distinct to a consumer group.)

The relationship between _P_ and _M_ is depicted in Figure 1.

<img src="https://rawgit.com/obsidiandynamics/neli/master/figures/Figure%201.svg" width="70%" alt="Figure 1
"/>

**Figure 1: Partition mapping from _M_ to _P_**

Each process _p_ in _P_, now being a consumer of _C_, will maintain a vector _V_ of size identical to that of _M_, with each vector element corresponding to a partition in _M_, and initialised to zero. _V_ is sized during initialisation of _p_, by querying the brokers of _M_ to determine the number of partitions in _M_, which will remain a constant. (As opposed to elements in _P_ and _R_ which may vary dynamically.) This implies that _M_ may not be expanded while a group is in operation.

Upon receipt of a message _m_ from _C_, _p_ will assign the current machine time as observed by _p_ to the vector element at the index corresponding to _m_'s partition index.

Assuming no subsequent partition reassignments have occurred, each _p_'s vector comprises a combination of zero and non-zero values, where zero values denote partitions that haven't been assigned to _p_, and non-zero values correspond to partitions that have been assigned to _p_ at least once in the lifetime of _p_. If the timestamp at any of vector element _i_ is _current_ &mdash; in other words, it is more recent than some predefined constant threshold _T_ that lags the current time &mdash; then _p_ is a leader for _M<sub>i</sub>_. If partition assignment for _M<sub>i</sub>_ is altered (for example, if _p_ is partitioned from the brokers of _M_, or a timeout occurs), then _V<sub>i</sub>_ will cease incrementing and will eventually be lapsed by _T_. At this point, _p_ must no longer assume that it's the leader for _M<sub>i</sub>_.

Ownership of _M<sub>i</sub>_ still requires a translation to a role assignment, as the number of roles in _R_ may vary from the number of partitions in _M_, and in fact, may do so dynamically without prior notice. To determine whether _p_ is a leader for role _R<sub>j</sub>_, _p_ will compute _k = j mod size(M)_ and check whether _p_ is a leader for _M<sub>k</sub>_ through inspection of its local _V<sub>k</sub>_ value.

Where _size(M) &gt; size(R)_, ownership of a higher numbered partition in _M_ does not necessarily correspond to a role in _R_ &mdash; the mapping from _R_ to _M_ is injective. If _size(M) &lt; size(R)_, ownership of a partition in _M_ corresponds to (potentially) multiple roles in _R_, i.e. _R_ &rarr; _M_ is surjective. And finally, if _size(M) = size(R)_, the relationship is purely bijective. Hence the use of the modulo operation to remap the dynamic extent of _R_ for alignment with _M_, guaranteeing totality of _R_ &rarr; _M_. 

**Note**: Without the modulo reduction, _R_ &rarr; _M_ will be partial when _size(M) &lt; size(R)_, resulting in an indefinitely dormant role and violating the _liveness_ property of the protocol.

Under non-exclusive leadership, the value of _T_ is chosen such that _T_ is greater than the partition reassignment threshold of the broker. In Kafka, this is given by the property `session.timeout.ms` on the consumer, which is 10 seconds by default &mdash; so _T_ could be 30 seconds, allowing for up 20 seconds of overlap between successive leadership transitions. In other words, if the partition assignment is withdrawn from an existing leader, it may presume for a further 20 seconds that it is still leading, allowing for the emerging leader to take over. In that time frame, one or more roles may be fulfilled concurrently by both leaders &mdash; which is acceptable _a priori_. (In practice, 10 seconds is too long for HA systems that approach continuous availability; smaller values such as 100 milliseconds are more suitable.)

There is no hard relationship between the sizing of _M_, _R_ and _P_; however, the following guidelines should be considered:

* _R_ should be at least one in size, as otherwise there are no assignable roles.
* _M_ should not be excessively larger than _R_, so as to avoid processes that have no actual _role_ assignments in spite of owning one or more partitions (for high numbered partitions). When using Kafka, this avoids the problem when `partition.assignment.strategy` is set to `range`, which happens to be the default. To that point, it is strongly recommended that the `partition.assignment.strategy` property on the broker is set to `roundrobin`, so as to avoid injective _R_ &rarr; _M_ mappings that are asymmetric and poorly distributed among the processes.
* _M_ should be sized approximately equal to the steady state (anticipated) size of _P_, notwithstanding the fact that _P_ is determined dynamically, through the occasional addition and removal of deployed processes. When the size of _M_ approaches the size of _P_, the assignment load is shared evenly among the constituents of _P_.

It is also recommended that the `session.timeout.ms` property on the consumer is set to a very low value, such as `100` for rapid consumer failure detection and sub-second rebalancing. This requires setting of `group.min.session.timeout.ms` on the broker to `100` or lower, as the default value is `6000`. The `heartbeat.interval.ms` property on the consumer should be set to sufficiently small value, such as `10`.


# Further considerations
## Topic reuse for independent role-process assignments
For the sake of clarity and, more importantly, to avoid failure correlation, it is highly recommended to use a dedicated topic for each NELI group, configured in accordance with the group's needs. However, under certain circumstances it may be more appropriate to share a topic across multiple NELI groups. For example, the broker might be offered under a SaaS model, whereby the addition of topics (and partitions) incurs additional costs.

By using a different consumer group ID in a common topic _C_ &mdash; in effect a different NELI group &mdash; the same set of partitions _M_ can be exploited to assign a different set of roles _R'_ to the same or a different set of processes _P_ or _P'_, with no change to the protocol.

When reusing a brokered topic across multiple NELI groups, it is recommended to divide the broadcast frequency by a factor commensurate with the number of NELI groups. In other words, if processes within a single NELI group broadcast with a frequency _F_ to all partitions in _C_, then adding a second NELI group should change the broadcast frequency to _F/2_ for both groups.

## Exclusivity of role assignment
It can also be shown that the non-exclusivity property can be turned into one of exclusivity through a straightforward adjustment of the protocol. In other words, the at-least-one leader assignment can be turned into an at-most-one (turning NELI into ELI). This would be done in systems where non-exclusivity cannot be tolerated.

The non-exclusivity property is directly controlled by the liberal selection of the constant _T_, being significantly greater than the broker's partition reassignment threshold (the session timeout) &mdash; allowing for leadership overlap. Conversely, if _T_ is chosen conservatively, such that _T_ is significantly less than the reassignment threshold, then the currently assigned leader will expire prior to the assignment of its successor, leaving a gap between successive assignments.

A further complication that arises under exclusive mode is the potential discrepancy between the true and the observed time of message receipt. Consider a scenario where two processes _p_ and _q_ contend for _M<sub>i</sub>_ and _p_ is the current assignee. Furthermore, _p_'s host is experiencing an abnormally high load and so _p_ is subjected to intermittent resource starvation. The following sequence of events are conceivable:

1. _p_ receives messages from _M<sub>i</sub>_ and buffers them within the client library.
2. _p_ is preempted by its host's scheduler as the latter reallocates CPU time to a large backlog of competing processes.
3. The session timeout lapses on the broker and _M<sub>i</sub>_ is reassigned to _q_.
4. _q_ begins receiving messages for _M<sub>i</sub>_, and assumes leadership.
5. _p_ is eventually scheduled, processes the message backlog and assigns a local timestamp in _V<sub>i</sub>_, thereby still believing that it is the leader for _M<sub>i</sub>_, for a further time _T_.

This is solved by using a broker-supplied timestamp in place of a local one. As publishing of a message _pub(m)_ is causally ordered before its consumption _con(m)_, i.e._pub(m) ⟼ con(m)_, then _BTime(pub(m)) &le; BTime(con(m))_, where _BTime_ denotes the broker time and is monotonically non-decreasing. (We use Lamport's [2] well-accepted definition of causality.) By using the broker-supplied publish timestamp, _p_ is taking the most conservative approach possible. However, the role check on _p_ still happens using _p_'s local time, comparing with the receipt of _m_ captured using the broker's time, and there is no discernible relationship between _BTime(pub(m))_ and _CTime(pub(m))_, where _CTime_ is the consumer's local time. 

Addressing the above requires that the clocks on the broker and the consumer are continually synchronised to some bounded error ceiling _e_, such that _∀t (|BTime(t) - CTime(t)| &le; e)_. Under this constraint, for a partition ownership check relating to _m_ for a threshold _T_ at time _t_, then it can be trivially shown that _∀m,t (isOwner(partitionOf(m), t, T) &rarr; t - CTime(pub(m)) &le; T + e)_. In other words, if the ownership check passes, then _m_ was published no longer than _T + e_ ago. It follows that if _T + e_ are both chosen so that their sum is smaller than the broker's session timeout time, then a positive result for a local leadership check will always be correct; a false positive will not occur providing that all the other assumptions hold and that the broker always honours the session timeout before instigating a partition reassignment.

Of course, the last assumption is not necessarily true. The broker may use other heuristics to trigger reassignment. For example, the broker's host OS may close the TCP socket with the consumer and thereby expedite the reassignment on the broker. The closing of the pipe might not be observed for some time on _p_'s host. Under this scenario, _p_ may falsely assume that it is still the leader. Assuming the broker has no minimum grace period, then this could be solved by observing a grace period on the newly assigned leader _q_, such that _q_ upon detection of leadership assignment, will intentionally return false from the leadership check function for up to time _T_ since it has implicitly acquired leadership. This gives _p_ sufficient time to relinquish leadership, but extends the worst case transition time to _2T_.

Another, more subtle problem stems from the invariant that the check for leadership must _always_ precede an operation on the critical region. (This is true for every system, irrespective of the exclusivity protocol used.) It may be possible, however unlikely, that the check succeeds, but in-between testing the outcome of the check and entering the critical region, the leadership status may have changed. In practice, if _T_ is sufficiently lower than the session timeout, the risk of this happening is very low. However, in non-deterministic systems with no hard real-time scheduling guarantees, one cannot be absolutely certain. Furthermore, even if the protocol is based on sound reasoning or is formally provable, it will only maintain correctness under a finite set of assumptions. In reality, systems may have defects and may experience multiple correlated and uncorrelated failures. So if exclusive mode is required under an environment of absolute assurance, further mechanisms &mdash; such as fencing &mdash; must be used to exclude access to critical regions.

## Use in continuously available systems
By the term _non-exclusive leader_ it is meant that at least one leader may be assigned; it shouldn't be taken that the assigned leader is actually functioning, network-reachable and is able to fulfill its role at all times. As such, single-group NELI cannot be used directly within a setting of strictly continuous availability, where at least two leaders are _always_ required. Of course, the thresholds used in NELI can be tuned such that the transition time between leaders is minimal, at the expense of work duplication. However, while being arbitrarily responsive (near-continuous), this doesn't formally satisfy continuous availability guarantees, which assume zero downtime under a finite set of assumptions (for example, at most one component failure is tolerated).

In a continuously available system, two or more disjoint (non-overlapping) NELI process groups _P1_ and _P2_ (through to _PN_, if necessary) may be used concurrently on the same set _M_ (or a different set _M'_, hosted on an independent set of brokers) and a common _R_, such that any _R<sub>i</sub>_ would be assigned to a member in _P1_ and _P2_, such that there will be at least two leaders for any _R<sub>i</sub>_ at any point in time &mdash; one from each set. Depending on the value of _T_, there may be three leaders during a transition event in any of the groups _P1_ or _P2_, assuming that process failures in _P1_ and _P2_ are uncorrelated.

Furthermore, it is prudent to keep the process sets _P1_ and _P2_ not only disjoint, but also deployed on separate hosts, such that the failure of a process in _P1_ will not correlate to a failure in _P2_, where both failed processes may happen to share role assignments.


# References
[1] K. Birman, "Reliable Distributed Systems", Springer. 2005.

[2] L. Lamport, "Time clocks and the ordering of events in a distributed system", Communications of the ACM, vol. 21, pp. 558-565, July 1978.

[3] F. Junqueira, B. Reed, M. Serafini, "Zab: High-performance broadcast for primary-backup systems" Proc. 41st Int. Conf. Dependable Syst. Netw. pp. 245-256 June 2011.

[4] D. Ongaro, J. Ousterhout, “In Search of an Understandable Consensus Algorithm” in USENIX Annual Technical Conference, 2014.