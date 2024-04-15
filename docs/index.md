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
[Understanding Paxos Using Three Simple Rules](paxos/understand-paxos.md)<br>
