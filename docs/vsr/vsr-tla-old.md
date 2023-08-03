---
title: "TLA+ Spec For View Stamped Replication"
date: 2023-07-03T13:11:18+05:30
draft: true
--- 

Replicas execute one of the above operations at any given time, accordingly the replicas maintain that status. 
This is mainly to avoid concurrent execution(like, say, reconfiguration during normal operation) to simply the overall algorithm. 
By disabling some of the operations while the algorithm is executing certain other operations there will be availability loss but it's 
a trade-off made to simplify the algorithm. VSR does include some other techniques so that overall availability is not severely affected, for example 
if new replicas are to be introduced, the  

Typically specifications use multiple TLA+ functions for representing the state of a distributed system. 
For example log, state, commit-no etc each indexed by replicas. This particular implementation uses a single TLA+ function named as `replicaState` which achieves better encapsulation & is indexed by replicas. This spec currently implements normal & view-change operations only & following safety/liveness properties. 


**Analysis:** 
Concurrent executions are hard to reason about on paper, also given that are innumerable possible executions TLA+ proves to be very useful. TLA+ spec serves as a ***completely tested prototype*** since every possible execution is tested for safety & liveness properties. Implementing the spec in TLA+ was particularly helpful in identifying the following scenarios & getting a better understanding of the replication protocol.

* Numerous concurrency issues, for example, collecting/resetting SVC & DVC message sets during nested view changes. The spec can used as starting point for reference during        actually implementation.  
* Commit-no can go backwards during view change. Such scenario can be addressed if state machine does dedupe using the commit-no **or** the VSR algorithm should ensure that commit-no grows monotonically.  
* Commit no can be greater than the log index. 
  Lets say there are 5 replicas, after getting responses from 2 replicas primary can advance the commit number. However up to 2 replicas might be lagging & might not have all the log entries up to the commit number being sent by the primary. 
