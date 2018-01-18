Non-Exclusive Leader Emergence in Highly Available Systems
===

# Overview
This text describes the Non-Exclusive Leader Emergence (NELE, pronounced 'Nelly') protocol built on top of a shared, partitioned ledger capable of partition balancing (such as Apache Kafka or Amazon Kinesis). The protocol yields a non-exclusive leader in a group of contending processes for one of a number of notional roles, such that each role has at least one leader assigned to it. The number of roles is dynamic, as is the number of contending processes. This protocol is useful in scenarios where &mdash;

* There are a number of roles that need fulfilling, and it's desirable to share this load among several processes (likely deployed on different hosts);
* While it's undesirable that a role is simultaneously filled by two processes at any point in time, the system will continue to function correctly. In other words, this may create duplication of work but cannot cause harm (the _safety_ property);
* Availability of the system is imperative; it's required that at least one leader is assigned to a role at all times so that the system as a whole remains operational and progress is made on every role (the _liveness_ property);
* The number of processes and roles is fully dynamic, and may vary on an _ad hoc_ basis. Processes and roles may be added and removed without reconfiguration of the group or downtime. This accommodates rolling deployments, scaling groups, and so forth;
* The use of a dedicated Group Membership Service (GMS) is deemed unviable, and where an alternate primitive is sought. Perhaps the system is deployed in an environment where a robust GMS is not natively available, but other capabilities that may internally utilise a GMS may exist. Kinesis in AWS is one such example.

**Note**: The term _emergence_ is used in favour of _election_ to communicate the notion of partial autonomy in the decision-making process. Under NELE, leaders aren't chosen directly, but through other phenomena that are observable by the affected group members, allowing them to infer that new leadership is in force. The protocol is also eventually consistent, in that members of the group may possess different views, but these views will invariably converge. This is contrary to a conventional GMS, where leadership election is the direct responsibility of the GMS, and is communicated to the affected parties.


# Practical implications
Consider a system for dissemination of financial pricing data to multiple subscribers. If this is a time-critical service, then it's imperative that prices are always provided in a timely manner and the downtime of any price provider is to be avoided. A traditional high availability (HA) setup with fail-over provides continuity of price data; however, gaps during the fail-over process will be observed. A simple solution is to introduce active-active redundancy; however, this leads to duplication of pricing information. 

In the above scenario, a non-exclusive leader solves the availability problem without introducing excessive data duplication. (In fact, duplication may only be observable during a leadership transition.) Furthermore, non-exclusive leadership may be used as a load-balancing technique, such that one leader is chosen for each of the independent financial instruments for which a stream of pricing data exists. So a distributed cluster of processes may share the load of publishing price streams without overlapping under normal conditions; no minority subset of processes is responsible for publishing of the majority of data streams.


# Protocol definition
A centrally-arbitrated topic _C_ is established with set of partitions _M_. (Assuming _C_ is hosted by Kafka or Kinesis.) A set of discrete roles _R_ is available for assignment, and a set of processes _P_ contend for assignment of roles in _R_. The number of elements in the set _M_ may vary from that of _R_ which, in turn, may vary from _P_.

Each process in _P_ continually publishes a message on all partitions in _M_ (each successive message is broadcast a few seconds apart). The message has no key or value; the producing process explicitly specifies the partition number for each published message. As each process in _P_ publishes a message to _M_, then each partition in the set _M_ is continually subjected to messages from each process. Corollary to this, for as long as at least one process in _P_ remains operational, there will be at least one message continually published in each partition in _M_. Crucial to the protocol is that no partition may 'dry up'.

Each process _p_ in _P_ subscribes to _C_ within a common, predefined consumer group. As per Kafka's partition assignment rules, a partition will be assigned to at most one consumer. Multiple partitions may be assigned to a single consumer, and this number may vary slightly from consumer to consumer. Note &mdash; this is a fundamental assumption of NELE, requiring a broker that is capable of arbitrating partition assignments.

Each process _p_ in _P_, now being a consumer of _C_, will maintain a vector _V_ of size identical to that of _M_, with each vector element corresponding to a partition in _M_, and initialised to zero. _V_ is sized during initialisation of _p_, by querying the brokers of _M_ to determine the number of partitions in _M_, which will remain a constant. (As opposed to elements in _P_ and _R_ which may vary dynamically.) This implies that _M_ may not be expanded while a group is in operation.

Upon receival of a message _m_ from _C_, _p_ will assign the current machine time as observed by _p_ to the vector element at the index corresponding to _m_'s partition index.

Assuming no subsequent partition reassignments have occurred, each _p_'s vector comprises a combination of zero and non-zero values, where zero values denote partitions that haven't been assigned to _p_, and non-zero values correspond to partitions that have been assigned to _p_ at least once in the lifetime of _p_. If the timestamp at any of vector element _i_ is _current_ &mdash; in other words, it is more recent than some predefined constant threshold _T_ that lags the current time &mdash; then _p_ is a leader for _M<sub>i</sub>_. If partition assignment for _M<sub>i</sub>_ is altered (for example, if _p_ is partitioned from the brokers of _M_, or a timeout occurs), then _V<sub>i</sub>_ will cease incrementing and will eventually be lapsed by _T_. At this point, _p_ must no longer assume that it's the leader for _M<sub>i</sub>_.

Ownership of _M<sub>i</sub>_ still requires a translation to a role assignment, as the number of roles in _R_ may vary from the number of partitions in _M_, and in fact, may do so dynamically without prior notice. To determine whether _p_ is a leader for role _R<sub>j</sub>_, _p_ will compute _k = j mod size(M)_ and check whether _p_ is a leader for _M<sub>k</sub>_ through inspection of its local _V<sub>k</sub>_ value.

Where _size(M) &gt; size(R)_, ownership of a higher numbered partition in _M_ does not necessarily correspond to a role in _R_ &mdash; the mapping from _R_ to _M_ is injective. If _size(M) &lt; size(R)_, ownership of a partition in _M_ corresponds to (potentially) multiple roles in _R_, i.e. _R_ &rarr; _M_ is surjective. And finally, if _size(M) = size(R)_, the relationship is purely bijective. Hence the use of the modulo operation to remap the dynamic extent of _R_ for alignment with _M_, guaranteeing totality of _R_ &rarr; _M_. 

**Note**: Without the modulo reduction, _R_ &rarr; _M_ will be partial when _size(M) &lt; size(R)_, resulting in an indefinitely unassigned role and violating the _liveness_ property of the protocol.

Under non-exclusive leadership, the value of _T_ is chosen such that _T_ is greater than the partition reassignment threshold of the broker. In Kafka, this is given by the property `session.timeout.ms` on the consumer, which is 10 seconds by default &mdash; so _T_ could be 30 seconds, allowing for up 20 seconds of overlap between successive leadership transitions. In other words, if the partition assignment is withdrawn from an existing leader, it may presume for a further 20 seconds that it is still leading, allowing for the emerging leader to take over. In that time frame, one or more roles may be fulfilled concurrently by both leaders &mdash; which is acceptable _a priori_. (In practice, 10 seconds is too long for HA systems that approach continuous availability; smaller values such as 100 milliseconds are more suitable.)

There is no hard relationship between the sizing of _M_, _R_ and _P_; however, the following guidelines should be considered:

* _R_ should be at least one in size, as otherwise there are no assignable roles.
* _M_ should be sized equal to or less than _R_, so as to avoid processes that have no actual _role_ assignments in spite of owning one or more partitions (for high numbered partitions). When using Kafka, this avoids the problem when `partition.assignment.strategy` is set to `range`, which happens to be the default. To that point, it is recommended that the `partition.assignment.strategy` property on the broker is set to `roundrobin`, so as to avoid injective _R_ &rarr; _M_ mappings that are extremely asymmetric.
* _M_ should be sized approximately equal to the steady state (anticipated) size of _P_, notwithstanding the fact that _P_ is determined dynamically, through the occasional addition and removal of deployed processes. When the size of _M_ approaches the size of _P_, the assignment load is shared evenly among the constituents of _P_.

It is also recommended that the `session.timeout.ms` property on the consumer is set to a very low value, such as `100` for rapid consumer failure detection and sub-second rebalancing. This requires setting of `group.min.session.timeout.ms` on the broker to `100` or lower, as the default value is `6000`. The `heartbeat.interval.ms` property on the consumer should be set to sufficiently small value, such as `10`.


# Further considerations
## Exclusivity of role assignment
It can also be shown that the non-exclusivity property can be turned into one of exclusivity through a straightforward adjustment of the protocol. In other words, the at-least-one leader assignment can be turned into an at-most-one. This would be done in systems where non-exclusivity cannot be tolerated.

The non-exclusivity property is directly controlled by the liberal selection of the constant _T_, being significantly greater than the partition reassignment threshold. If, on the other hand, _T_ is chosen conservatively, such that T is significantly less than the reassignment threshold, then the currently assigned leader will expire prior to the assignment of its successor, leaving a gap between successive assignments.

## Topic reuse for independent role-process assignments
By using a different consumer group ID, the same set of partitions _M_ can be exploited to assign a different set of roles _R'_ to a different set of processes _P'_. In other words, there is no compelling need to establish a new topic for a different set of roles.

## Use in continuously available systems
By the term _non-exclusive leader_ it is meant that at least one leader may be assigned; it shouldn't be taken that the assigned leader is actually functioning, network-reachable and is able to fulfil its role at all times. As such, single-group NELE cannot be used directly within a setting of strictly continuous availability, where at least two leaders are _always_ required. Of course, the thresholds used in NELE can be tuned such that the transition time between leaders is minimal, at the expense of work duplication. However, while being arbitrarily responsive (near-continuous), this doesn't formally satisfy continuous availability guarantees, which assume zero downtime under a finite set of assumptions (for example, at most one component failure is tolerated).

In a continuously available system, two or more disjoint (non-overlapping) NELE process groups _P1_ and _P2_ (through to _PN_, if necessary) may be used concurrently on the same set _M_ (or a different set _M'_, hosted on an independent set of brokers) and a common _R_, such that any _R<sub>i</sub>_ would be assigned to a member in _P1_ and _P2_, such that there will be at least two leaders for any _R<sub>i</sub>_ at any point in time -- one from each set. Depending on the value of _T_, there may be three leaders during a transition event in any of the groups _P1_ or _P2_, assuming that process failures in _P1_ and _P2_ are uncorrelated.

Furthermore, it is prudent to keep the process sets _P1_ and _P2_ not only disjoint, but also deployed on separate hosts, such that the failure of a process in _P1_ will not correlate to a failure in _P2_, where both failed processes may happen to share role assignments. 

Alternatively, if _P1_ and _P2_ are to share their runtime environment and brokers for _M_, a custom assignment strategy must be implemented, such that a partition in _M_ can never be assigned to the same host across two different consumer groups. (To the best of our knowledge custom assignment strategies are not yet available in Kafka; however, a [client-side assignment proposal](https://cwiki.apache.org/confluence/x/foynAw) had been drafted in 2015.)
