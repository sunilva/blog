---
title: Comparing VSR And Raft Algorithms 
hide:
  - navigation
---

### Introduction

View Stamp Replication(VSR) & Raft are replication algorithms for implementing fault replicated state machines(basis for building other
distributed applications like storage, database etc). 

VSR was first published in <todo> independent of Paxos, however for this discussion we will be considering VSR revisited.  

Both assume asynchronous model in which no assumptions are made w.r.t time, crash failure & network in which messages can be 
delivered out-of-order, lost, duplicated & delayed for arbitrary time but delivery is guaranteed if retried indefinitely i.e network is reliable. 
VSR doesn't need disk & assumes failure independence of the hosts.

Raft's design is aimed as Understandability.

| VSR         | Raft        |
|-------------|-------------|
| Primary     | Leader      |
| Backup      | Follower    |
| View        | Term        |
| Epoch       | -NA-        |
| Replication | Normal      |
| Election    | View Change |


### Replication
One of the hosts in the system is elected a leader which receives requests from clients & replicates the 
requests to other hosts called followers.

----
Raft sends two entries viz. current & previous while replicating entries to follower, foll
Raft sends empty append entry requests as heart beats to follower.

----
VSR sends a single entry to the backup replicas, if backup identifies it is missing entries 
the it does a GET-STATE operation to fetch missing entries

### Leader election

#### Raft
Raft uses voting mechanism to elect a new leader. A follower increments it's term, becomes a candidate 
and requests a votes from the followers. If any candidate receives a quorum of votes then it transitions to it's 
state to become leader for the new term. 

<todo> State diagram for leader election is shown below: 
<todo> timeline view

Safety: 
    A quorum of votes are required to be a leader for a given term. 

    Replication requests carry the current term, if follower's term doesn't match the leader the request is rejected 
    by the follower.  If election fails due to split votes then election is started for next term abandoning the 
    term for which election was in progress.

Liveness:
    Election can if votes get's split across more than one candidate & nobody receives a majority of the votes(quorum). 

    Raft uses randomization technique so as to avoid multiple followers initiating election at about the same time
    resulting in split votes. 

#### VSR
VSR uses a three phase protocol to elect a new primary, the safety condition being: "No committed entry should be lost" 

* Phase-1(SVC): 
    A quorum of replicas agree to stop participating in replication process no new entries get added to logs & commit
    doesn't happen. 
* Phase-2(DVC): 
    The new primary gets logs from a quorum of replicas, the new primary determines log to be used for initializing 
    the next view.
* Phase-3(SV):  
    The new primary initializes all the replicas with log determined in phase-2 & starts replication process for the 
    next view.    

StartViewChange(SVC): 
    Upon timeout expiry one or more backups increment the view & start SVC broadcast messaging. 
    If a replica receives a quorum of SVC messages then it sends DVC message to the next primary.
DoViewChange(DVC): 

StartView(SV): 
    Start view message initializes the log of the backups at the start of the next view 
    with log & commit number determined during the DVC phase.

Safety: 
    Leader election is safe as quorum of hosts need to agree on the next view. 
    As quorum of replicas agree on the next view, requests from previous primary(if it hadn't crashed but merely 
    slowed down temporarily) will be ignored at least one replica which has moved to next view. 
    Hence previous primary will not be able to commit any request as it can't replicate to a quorum of replicas.

Liveness: 
    If the chosen host is down then term gets incremented & another round of SVC messaging is triggered to choose 
    another host as a new leader. SVC messaging should continue indefinitely until a quorum of replicas are contacted.

    As hosts co-ordinate rather than compete to elect next leader FLP has minimal impact on VSR as compared to Raft. 
    The chosen host has to be done if leader election has to fail. 

### Commit
* Leader doesn't overwrite

### Recovery 
 
Raft persists every request as it gets replicated hence the mechanism required in recovering a failed follower is minimal. 
Further Leader assists the follower to recover. 

VSR doesn't use disk. In practical implementations VSR persists client request to the disk in the background. 
VSR restores it's state from other hosts in the system. For safety reason it contacts a quorum of hosts which 
needs to consist leader as well, to fetch the latest state.

### Reconfiguration


### Conclusion

VSR relies only a single axiomatic rule that commit of entry is safe if it is replicated on quorum of replicas.
Raft follows the philosophy follow the leader, intelligence is concentrated on leader.

State space reduction should be aim of algorithm design 
Reducing the amount of mechanism involved in the algorithm is helpful for practical implementation.

-----
Replication:

Leader election:

Recovery: 

Reconfiguration: 
    Raft reconfiguration is very simple
-----

While VSR uses simple axiomatic rules for commit guarantee, Raft doesn't 
Raft seems like an engineering solution coming out of industry while VSR presentation is more academic in nature.
