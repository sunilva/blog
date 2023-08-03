---
hide:
  - navigation
  - toc
---

# ***Slow Host*** 

A blog on distributed systems discussing various topics like:<br>*Consensus, replication algorithms, commit protocols, transactions, consistency models, time, formal verification etc.*

The two fundamental problems concerning distributed systems are <u>***concurrency & failure***</u>.<br>
As far as failure is concerned a host need not necessarily crash just by being slow it can prevent consensus
from being achieved. <br>This is the famous [FLP result](https://www.cs.cornell.edu/courses/cs614/2007fa/Slides/FLP_and_Paxos.pdf) (*as it is impossible to distinguish between a slow host & crashed host in an asynchronous system*). Hence the name of this blog.

### **Posts**
#### 2023
##### Aug 
*2023-08-02* - [Understanding Paxos Using Three Simple Rules](blog/paxos/paxos.md)<br>

##### Aug
*2023-08-03* - [Analyzing View Stamp Replication](blog/vsr/analyze-vsr.md)<br>
*2023-08-03* - [TLA+ Spec for View Stamp Replication](blog/vsr/vsr-tla.md)<br> 

### **Upcoming**
1. TLA+: A Design Tool for Distributed Systems
2. Comparing VSR & Raft Replication Algorithms
