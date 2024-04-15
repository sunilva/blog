---
title: "Understanding Paxos With Three Simple Rules"
hide:
  - navigation
---

### Introduction

#### Fault Tolerant System
Consider a client server system, clients can send requests to the server. The server changes it’s state as it serves the client requests. For instance, a seat reservation system reduces the number of available seats & sends a confirmation to the client. The server can be abstracted as a state machine. 

A state machine(SM) starting in certain state(*some number of available seats*) transitions to new state(*reduced number of available seats*) by executing a command taken as a input(*client request*) while producing an output(*reservation confirmation*). 

If the next state of a state machine depends completely on the input being executed then it is said to be deterministic. A set of deterministic state machines will behave identically if they^[1](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)^: 

- start in the same initial state.
- execute the same set of commands.
- excute the commands in the same sequence.

Such a set of state machines is called a replicated state machine(RSM). A RSM can sustain some of the server(henceforth called process) failures and hence provide fault tolerance. As you can see the group of servers constituting RSM need to agree on the set & order of the commands to be executed, that brings us to consensus.

#### Problem definition
Consensus is defined as a group of processes agreeing on a single value(*i.e client request to be executed*). 
Consensus needs to address the following three conditions:

* Agreement: A single value is chosen.
* Validity: The value agreed upon value is actually proposed(*this is to avoid trivial solutions, all processes can simply decide a fixed value, say 42, no matter what).
* Termination: Eventually a value gets chosen.

The first two are safety properties & last one is liveness property^[2](https://en.wikipedia.org/wiki/Safety_and_liveness_properties)^.

If a group of processes are able to agree on single value(*basic or single decree paxos*) then the solution can be extended to agree on a sequence of values(*multi paxos*) which will be useful in realizing a RSM. 

#### Environment
An algorithm needs to consider the environment in which it is supposed to work.
The processes are connected by a network & communicate with each other by exchanging messages, the processes & network together are referred to as a system. 

* *Processes:* 
    - A process can take arbitrary time to respond back to a message. 
    - A Process can crash & restart. Since consensus is impossible if a process forgets everything after a crash, 
      it is assumed that processes can use stable storage to remember certain information across a crash. 

* *Network*: 
    - Messages can get arbitrarily delayed in the network. 
    - Messages can get duplicated, be delivered out-of-order or get lost. 
    - The network is considered to be reliable in the sense that if the sending process retries then message will be delivered, eventually. 

Since the processes & network don't provide any guarantees w.r.t time the system is said to be **asynchronous**.
The above conditions mean that the algorithm need to be devised without relying on **time** for it to be robust & work correctly(ensure safety).

!!! note "Failure Detection"
    *A fundamental problem in an asynchronous distributed system is that it is impossible to tell ***whether a process has crashed or slowed down***.*
    
    **Why can't failure be detected using a request-response mechanism ?**
        <br>The expected time to receive a reply for a ping request can't be bounded as, by definition, a process can take arbitrary time to respond back 
        & the network transit time delay for a message is also unbounded.

### Consensus
**Acceptors & proposers:**
The processes are divided into a set of proposers & acceptors. Proposers propose to acceptors in order to choose a value. 
It is possible for a single process to be an acceptor & proposer but for simplicity we will consider acceptors & proposers to be distinct.

**Single fixed proposer**:
A single proposer accepts requests, chooses one among them & proposes to all the acceptors. This solution doesn't work as it can't make any progress if the fixed proposer fails. 
If a new proposer is needed to step-in then it needs to do two things: 

* Ensure that the old proposer doesn't affect the system adversely if it comes back(the old one din't fail but slowed down, as discussed earlier it can't be said derterministically if a process has failed).
* Reconcile the current system state & accordingly propose a value to ensure safety.

**Multiple proposers**:
Multiple proposers accept requests & propose to the acceptors. Acceptors accept only once & a value is considered to be chosen if it gets accepted by a majority of the acceptors. Since any two majorities have atleast one acceptor in common & acceptors accept once - only a single value can be chosen. However this solution doesn't work if:

* No majority accepts a single value amidst multiple concurrent proposals.
* An acceptor in the majority dies after a value had been chosen.

Note that if there are **2N+1** acceptors then **N+1** or more acceptors form a majority, since majority acceptance is sufficient 
to choose a value, upto **N** acceptor failures can be sustained.

!!! note "Single vs Multiple Proposers"
    Paxos assumes multiple proposers & builds upon it dealing with two fundamental problems in distrbuted systems viz. **concurrency & failure**. **Concurrency** is hard to reason about while **failure** can't be ascertained in an asynchronous system, deterministically. Moreover a proposer or an acceptor may fail at any moment(the algorithm needs to be safe).

    Replication algorithms like Raft, View Stamped Replication(VSR) & Zookeeper Atomic Broadcast(ZAB) build upon single fixed proposer approach. They aren't 
    consensus algorithms but implement RSM.

==quorum trick: global state local knowledge, space time resolution, concurrecny resolution.==
### The Three Rules
#### <u>Rule One: Multiple Rounds</u>

Owing to **concurrency & failure** problems discussed in the previous section, more than one proposal may be required to achieve consensus. 

* If no majority is achieved owing to multiple simultaneous proposals then a new proposal needs to be tried to achieve consensus. 
* If an acceptor fails after a majority is achieved then new proposal should re-establish the majority for the same value. Accordingly acceptors should be allowed to accept more than one proposed values.  

**Proposal Numbers**: Since multiple proposals are involved they need to be numbered. Each proposal carries a proposal number & value being proposed. 
Paxos is based on two simple invariants viz. 

* Single value is proposed for a given proposal number. A proposal number can't be reused to propose a different value.
* Once consensus is achieved for a certain value then all subsequent proposals can achieve consensus only for that value. 

To meet condition one each proposer chooses proposal numbers from disjoint sets to avoid reuse across proposers & a proposer can simply store the last proposal number used by itself to prevent reuse by self. For condition two total order of proposal numbers is required(notice the word *subsequent*), also a two phase protocol discussed below becomes imperative. 

Proposal number is constituted using two integers ***< I,J >***. Each proposer is numbered uniquely & this constitutes the second part ***"J"*** of the proposal
number which will ensure that the proposal numbers are unique to the proposer & doesn't get reused across proposers. For instance proposer one will use 11, 21, 31 etc, while 
proposer two will use 12, 22, 32 etc & so on. The first part of proposal number ***"I"*** is a monotonically increasing number. A proposer, at the start of each new proposal, queries a quorum of acceptors to know about all past proposals & chooses a new ***"I"*** such that the new proposal number is greater than all the past proposals.

**Two phase protocol:** At the start of each new proposal, first a query is made to atleast a majority of acceptors 
to know the system state & then a value is considered for current proposal. The query phase is called **prepare** & propose phase is called **accept**. Restrictions are put on what value can be proposed which is discussed in the next two rules, this rule is more about multiple porposals & how proposal numbers need to be chosen. 

Note that there can multiple simultaneous ongoing proposals but no two proposals should achieve consensus for different values.

!!! note "Rule One: Mutliple proposals may be required to achieve consensus"
    ***<center><span style="color:blue">Due to concurrency & failure, more than one proposal may be needed to achieve consensus. A single value is 
    proposed in each proposal & proposals are totally ordered(those which start later have a higher proposal number).</span>***

#### <u>Rule Two: Kill The Past</u>
Consider the situation in the below diagram:
``` title="An acceptor common between two proposals hasn't accepted any value"

    |<---P1--->|            * N1 & N2: Process groups consisting of N acceptors each.
    +------+---+------+     * Proposals P1 & P2 have an acceptor "A" in common, P2 > P1.
    |  N1  | A |  N2  |     * "A" hasn't accepted a value from P1, A.value = nil.  
    +------+---+------+     * N1 & N2 don't have any other acceptor in common.
           |<---P2--->|    
```

1. Acceptor **"A"** which is common between proposals **P1** & **P2** hasn't accepted any value.
2. Since acceptors in **N1** are not involved in **P2**, it is not possible for proposal **P2** to know what value 
acceptors in **N1** have accepted(or will accept in future & achieve consenus).
3. **P2** can choose to propose any value, however it should ensure that a different value is not chosen by **P1**.
4. To deal with above situation, proposer for **P2** makes a prepare request & extracts a promise from acceptors in **N2** & **"A"** 
not to accept any proposal lesser than itself.
5. Hence proposal **P1** will not be able to achieve acceptance from a majority i.e **P2** just killed **P1**.
   Note that a quorum of acceptors (**N2** + **A**) have moved to a higher numbered proposal hence **P1** will not be able to achieve consensus.

Note: 

* There can be more than one acceptor common between two proposals.
* There could be more than one proposal before current proposal, all those proposals in which *<u>at least one common acceptor</u>* hasn't yet accepted a value will be killed by current proposal.

!!! note "Rule Two: Kill The Past"
    ***<center><span style="color:blue;">Extract a promise from acceptors participating in the current proposal not to accept any proposal having proposal number lesser than the current one. Once a promise has been extracted from a quorum, some of the ongoing proposals(in prepare or accept phase) which have a lesser proposal number can't achieve consensus.***</span>

#### <u>Rule Three: Past Is Prologue</u>

=== "Single Accepted Value From Previous Proposal"
      <center>**Note: Make sure you read next tab after reading this one.**</center>
      ```title="Acceptor A which is common between proposals P1 & P2 has accepted a value."

      |<---P1--->|            * N1 & N2: Process groups consisting of N acceptors each.
      +------+---+------+     * Proposals P1 & P2 have an acceptor "A" in common, P2 > P1.
      |  N1  | A |  N2  |     * "A" has already accepted a value from P1, A.value = P1.value.
      +------+---+------+     * N1 & N2 don't have any other acceptor in common.
             |<---P2--->|
      ```

      1. Acceptor **"A"** has accepted **"a"** as a value from **P1**.
      2. If a different value, say **"b"**, were to be chosen in proposal **P2** then: 
           * Acceptors in **N1** can complete the proposal **P1** which chooses **"a"**. 
             Note that proposal **P2** doesn't has any control over the acceptors in **N1** as they are not part of the proposal.
             It's also possible **P1** has already reached consensus.
           * Proposer for **P1** would deem **"a"** as chosen value & proposer for **P2** would deem **"b"** as chosen value.
           * Ultimately, though a majority has accepted **"b"** after **P2** completes, there would have been a <u>flip from **"a"** to **"b"**</u> 
             at some point in time if proposal **P1** completes before **P2**. 
      3. To avoid above situation, before proposing a value in proposal **P2**, the proposer tries to learn if any of the acceptors in **P2** had already 
         accepted a value from past proposal in the prepare phase. If so, then the previously accepted value is used for the current proposal **P2**.
      4. We are still not done, what if more than one common acceptors have accepted different values from different past proposals.
         A question that arises is **which value** should be considered for the current proposal? Continue to the next tab for answer.
         
=== "Multiple Accepted Values From Different Past Proposals"
      <center>**Note: Make sure you have read previous tab before reading this one.**</center>
      As per quorum property the current proposal will have one or more common acceptors in each of the past proposals.Hence, whenever a new proposal 
      starts it will able to know about all the past proposals(*< proposal-number, value-proposed >*) through prepare phase.  Given a past proposal & current proposal, the past proposals will fall into two categories:

      * At least one common acceptor hasn't accepted a value, such proposals will be killed following **Rule-2**.      
      * One or more common acceptors has accepted a value, such proposals will be in one of these states: ***dead, ongoing or consensus achieved***. 
        However, the current proposal can't know about the <u>status of the past proposals</u>. 
        Current proposal has to choose one value from the different values proposed in these past proposals preserving ***safety***.

      Lets begin from the start. Considering the very first two proposals, two things are possible: 

      * **P2** proposal kills **P1**, the value proposed in **P2** & **P1** proposals can differ(following Rule-2). <br>We get the sequence: =="dead-ongoing"==.
      * **P2** chooses the same value as **P1**(as discussed in previous tab), both **P2** & **P1** are ongoing. <br>We get the sequence: =="ongoing-ongoing"==.
      
      For the third proposal, the following below given scenarios are possible in the prepare phase when a query is made.
      ```
          a               b               c               d
          ------------------------------------------------------------    
          dead-ongoing    dead-ongoing    dead-ongoing    dead-ongoing
             ^       ^       ^                       ^ 

        ^: Indicates acceptors in current proposal respond with values accepted in the past proposals when a 
           prepare request is made for current proposal.

        a: The accepted values correspond to dead & ongoing proposal. 
           Here value corresponding to ongoing proposal needs to be considered for current proposal 
           to preserve safety.
        
        b: The accepted value corresponds to dead proposal. Since no acceptor participating in the current proposal 
           has accepted a value from the ongoing proposal, it will get killed. 
           So it is safe to consider the value corresponding to the dead proposal.
        
        c: The accepted value corresponds to an ongoing proposal. 
           It's safe to choose this value for current proposal.
        
        d: None of the accceptors in current proposal have accepted a value in past proposals. 
           All the past proposals(including already dead, aha) will get killed.
           The current proposal is free to choose any value from the wild for the current proposal. 
      ```
      When a new proposal **P3** starts, is it possible to get a sequence like: =="ongoing-dead"==. 
      Yes this is possible. However, as far as the values(for safety we are concerned only with the value) are concerned it is still equivalent to =="ongoing-ongoing"==.
      Generalizing, it is easy to see that we need to consider a sequence like: =="ddddooooo"==, dead proposals followed by ongoing proposals.
      A sequence like =="ooddodd"== is equivalent to =="ooooooo"== as the dead proposals will have the same value as the ongoing proposals.

      Given a sequence like =="ddddooooo"== & if more than one acceptor in current proposal had already accepted a value in past proposals, then it is safe to consider the value corresponding to the **highest numbered past proposal**. If the highest numbered proposal corresponds to an ongoing proposal, the sequence gets extended with another ongoing proposal. However, if highest numbered proposal cooresponds to a dead proposal then all the ongoing proposals will get killed & the new sequence will have only one ongoing proposal.

      Once consensus is achieved only that value can be used in all future proposals as all future proposals will have at-least one common acceptor with
      the proposal which achieved consensus. This holds true even if there are upto **N** failures.

      Also, note that acceptors need not have to remember all the values proposed in the past proposals, the one corresponding to the highest numbered proposal is sufficient.
      
    !!! note "Rule Three: Past Is Prologue"
        ***<center><span style="color:blue;">If more than one acceptor participating in the current proposal had already accepted a value from different past proposals 
        then use the value corresponding to the highest-numbered past proposal for the current proposal to ensure safety</span>.***

### Safety & Liveness

#### Safety
  Paxos algorithm is safe to proposer & acceptor failures, they may fail or become slow at any moment.
  Note that acceptors in the system & their total number should be known apriori.
  Paxos can sustain upto **N** acceptor failures given **2N+1** acceptors, if more than **N** acceptors fail then it can't make progress as majority is needed.
  However system remains safe & can continue operation once the acceptors restart. Paxos is safe even if there are multiple simultaneous proposals. 
  Any number of proposers may fail, or a proposal can simply be abandoned in the middle. 

  **Moment of consensus**: Consensus is achieved at the moment the last acceptor in the quorum accepts a value. At this moment neither an acceptor nor the proposer know that 
  consensus has been achieved. At this moment either the proposer or an acceptor may fail, however consensus can be achieved again with a new proposal but for the same value.
  It is easy to reason about proposer failure. If an acceptor had failed then new majority will need atleast one acceptor from the quorum which had achieved consensus earlier - thus past consensus value makes it's way to the subsequent proposals.

  Paxos is safe to message delay, drop, duplication & re-order. The sending & receiving of messages is just an additional detail, safety of the Paxos algorithm is 
  ensured by how proposers & acceptors handle the messages.

#### Liveness
Note that initially we stated that eventually a value should be decided by all correct processes. 
Under certain condition this is not possible, this is not a serious limitation though & can be circumvented for practical purposes.

**Rule-2** causes liveness issue, lets say an ongoing proposal is killed & new one starts. But then the current proposal can get killed by a future proposal 
& the process can continue endlessly. Hence a value never gets chosen. The scenario is illustrated in the diagram below.
There are two proposers & for simplicity a single acceptor(common between the two proposals) is being shown. It keeps on rejecting 
the proposals as it keeps getting higher numbered prepare requests before an accept request.

```title="Paxos liveness issue due to livelock"

P1 ---<1>---------[1,a]----<3>---------------[3,a]----      <n>: proposal request
        \           \       \                  \            [<proposal-no>, <value>]: accept request  
         \           \       \                  \           x: rejected
A1 ------<1>---<2>----x------<3>----x----<4>-----x---
               /                   /      / 
              /                   /      / 
P2 ---------<2>----------------[2,b]----<4>----------
```

The above liveness issue is attributed to FLP result, which states that consensus is not possible in an asynchronous system even if there is 
a single process failure(*which occurs in the nick of time thus preventing consensus from being achieved*).

??? "FLP in the context of Paxos" 
    Liveness issue discussed above is attributed to the famous FLP proof which asserts that consensus is **impossible even with a single faulty 
    process** in an asynchronous system. What FLP really means is that consensus is not possible **sometimes** under certain conditions. 
    Consider the below excerpts taken from the FLP paper.

    !!! warning ""
        **“window of vulnerability”** - an interval of time during the execution of the algorithm in which the delay or inaccessibility 
        of a single process can cause the entire algorithm to wait indefinitely.

    !!! warning ""
        the stopping of a single process at an **inopportune time** can cause any distributed commit protocol to fail to reach agreement. 
      
    The theorem is proved by ***delaying a message at a crucial point*** during the execution & thus steering the algorithm execution 
    away from consensus being achieved. The delaying of message is attributed to a faulty process - **crashed or slow process**. 
    And in an asynchronous system it's impossible to distinguish between slow process & crashed process. 
    
    This leads to the dilemma between two choices:

      * whether to wait(slow process) which can lead to indefinite block if the process has actually crashed.
      * proceed further(crashed process) by resetting the execution(disabling the current execution to ensure safety). 
        However, in this case, a future reset is imaginable & the process continues endlessly.

    ***In the case of Paxos, Rule-2 disables previous proposals as it starts a new proposal which can itself get disabled by a future proposal 
    as illustrated earlier.***

Paxos suggests to elect a single distinguished proposer called leader. If the leader is given sufficient time to complete both the phases(prepare & accept) & the system(proposer, acceptors & network) is working properly then consensus will be achieved. Note that the liveness issue doesn't disappear but it is cleverly diverted to leader election(where it prevails). Sloppy timeouts can be used so that there aren't many proposers at the same time.  

### Paxos Algorithm
The complete paxos algorithm is summarised below:

!!! info "Complete Paxos Protocol"
    ```
    Phase 1. (a) A proposer selects a proposal number n and sends a prepare
             request with number n to a majority of acceptors.
             
             (b) If an acceptor receives a prepare request with number n greater
             than that of any prepare request to which it has already responded,
             then it responds to the request with a promise not to accept any more
             proposals numbered less than n and with the highest-numbered proposal
             (if any) that it has accepted.  
    
    Phase 2. (a) If the proposer receives a response to its prepare requests
             (numbered n) from a majority of acceptors, then it sends an accept
             request to each of those acceptors for a proposal numbered n with a
             value v, where v is the value of the highest-numbered proposal among
             the responses, or is any value if the responses reported no proposals.
             
             (b) If an acceptor receives an accept request for a proposal numbered
             n, it accepts the proposal unless it has already responded to a prepare
             request having a number greater than n.
    ```
    !!! note   
        The quorum in the accept phase need not necessarily be the same as the one during prepare phase.
        
        Since safety is guaranteed by rule-2 & rule-3 during prepare phase, it doesn't matter which 
        acceptors are part of the quorum during accept phase. However, checkout this 
        [potential bug](https://brooker.co.za/blog/2021/11/16/paxos.html).

### Replicated State Machine(RSM)

Practical systems require that a set of processes agree on a series of values instead of just one value. The decided values are added to a log 
with each decision going into an index in the log & since all acceptors will have the same log it is referred to as replicated log, shown below.

```
        *                                                   x
        |                                                   |     
    *   x   *       *                                       x   *   x
    |   |   |       |                                       |   |   |
    x   x   x   *   x                               *   *   x   x   x   *   *
    |   |   |   |   |                               |   |   |   |   |   |   |
  +---+---+---+---+---+                           +---+---+---+---+---+---+---+ 
  | 1 | 2 | 3 | 4 | 5 |                           | 1 | 2 | 3 | 4 | 5 | 6 | 7 |  
  +---+---+---+---+---+                           +---+---+---+---+---+---+---+ 
                                                  <--L1-->|<~~~~~~~~~>|<--L2-->    
                                                          |<-------L2-------->|                

  Fig.a: Multiple instances of Paxos,             Fig.b:     
  each instance chooses value for one slot.       <---->: Leader is established
  Vertical lines indicate Paxos run(proposals)    <~~~~>: Leader has died, multiple  
                                                          proposers are vying for leadership. 
                                                          Holes are possible, 4 is accepted 
                                                          while 3 & 5 aren't accepted. 
                                                          Once L2 establishes it's leadership it will 
                                                          consolidate the log(partially accepted values) & 
                                                          continue being a sole leader starting from index 6.   

  x: partially accepted value
  *: consensus achieved
```

For realizing a replicated log multiple instances of paxos can be run as shown in the above diagram on the left side. 
However this is not an efficient implementation. In a more practical implementation a single proposer(called leader) can issue 
just accept requests for multiple slots as shown in the above diagram on the right side. 
Leader is selected through leader election & leader maintains authority(single leader exists) using heart beat mechanism, 
other proposers won't vie for leadership as long as they receive heart beat messages. 

**committing a value:**
A replicated log basically provides a fault tolerant layer using which distributed applications(like db, object storage etc) can be built. 
Once a value is chosen it can be used by application layer - this is called commit, note that all the values in the log need to be committed in a serial order. 
Since only the leader knows consensus was achieved the commit index also needs to be communicated to the acceptors.
The leader can propose values for multiple slots simultaneously, so it is possible a proposal for greater numbered index to get accepted before a proposal 
for lesser numbered index is accepted i.e holes are possible in the log. In this case the greater numbered index can't be committed till values for all 
lesser numbered indices are also committed.

**handling leader failure:**
The leader can fail while proposing values so there can be multiple indices which are partially accepted. A new leader needs to be elected, 
the elected leader runs consensus(both propose & accept) for all slots greater than the commit index. There can be multiple proposals for the partially accepted 
slots if elected leader dies soon after being elected in the middle of proposal.

Once a stable leader is established & it resolves all the partially accepted indices. There after it can continue with normal operation wherein it would just 
issue accept & commit requests.

Note that multi-paxos is note well documented, the above description is only rough sketch. Practical implementation would have to consider many 
things adding/removing or increasing/decreasing the no. of acceptors in system, materializing the log to application state, application state
transfer among acceptors for recovery in order to bring the acceptor to latest system state etc. 

### Further Reading

**Basic Paxos:**
Leslie Lamport first developed Paxos as voting algorithm which uses shared memory model. If you are still not clear with above explanation you can check 
out the voting algorithm here (you will get to learn TLA+ as well):

* [The Paxos algorithm or how to win a Turing Award. Part 1.](https://www.youtube.com/watch?v=tw3gsBms-f8)
* [The Paxos algorithm or how to win a Turing Award. Part 2.](https://www.youtube.com/watch?v=8-Bc5Lqgx_c)

The above two videos are highly recommended(TLA+ crash course is a bonus).

You can also check out: 
Lampson's [How to Build a Highly Available System Using Consensus](https://www.microsoft.com/en-us/research/uploads/prod/1996/10/Acrobat-58-Copy.pdf).
& [Paxos lecture (Raft user study)](https://www.youtube.com/watch?v=JEpsBg0AO6o)

**Multi-Paxos & Reconfiguration:**
Two things complicate Paxos viz. need to a series of values for practical applications & changing the acceptors(membership) in the system.
Multi-paxos is not well documented, one can look into the formal specification of [multi-paxos](https://arxiv.org/abs/1606.01387).  

In distributed systems it is a given that hosts will fail ultimately. 
Hence the group of acceptors need to be changed: either replace an acceptor eventually or increase/decrease the no. of acceptors in the group 
to alter fault tolerance. To understand more about the same you can see 
[Paxos made moderately Complex](https://www.cs.cornell.edu/courses/cs7412/2011sp/paxos.pdf) & [Vertical Paxos](https://lamport.azurewebsites.net/pubs/vertical-paxos.pdf). 

**Distributed Lock:**
One practical application of paxos is to hold distributed locks in a fault tolerant manner: 
[Chubby service](https://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf) & [Zoo keeper](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf).

**Practical Implementation:**
For challenges concerning production grade implementation you can check out [Paxos made live](https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf).

Perhaps, basic Paxos is simple to understand, it is the combination of reconfiguring a paxos group, multi-paxos, implementing a 
service like distributed locking using consensus & operating a production grade system is what poses challenges.  

**Replicated log:**
Apart from Paxos, View Stamp Replication(VSR), ZAB & Raft are replication based algorithms which can be used for realizing a replicated log.  

A replicated log is considered to be a fundamental higher level primitive for building distributed systems. 
Jay Kreps has written about it here in a [blog post](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) & later wrote a [short book](https://www.oreilly.com/library/view/i-heart-logs/9781491909379/). 
[Corfu](https://www.usenix.org/conference/nsdi12/technical-sessions/presentation/balakrishnan) talks about implementing various distributed systems 
like transactional KV store, databases etc & AuroraDB views [log as the database](https://assets.amazon.science/dc/2b/4ef2b89649f9a393d37d3e042f4e/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases.pdf). 

Hence Paxos is a very fundamental algorithm for distributed systems. 

**Variants:** For different variants of paxos you can checkout: https://paxos.systems/variants/

**Trivia:**
A very nice history of consensus is published [here](https://betathoughts.blogspot.com/2007/06/).
It would be interesting to read about how Lamport came up with [Paxos algorithm]((https://lamport.azurewebsites.net/pubs/pubs.html#lamport-paxos)) & it's initial reception. A subtle bug referred to [here](https://lamport.azurewebsites.net/pubs/pubs.html#paxos-simple) due to misinterpretation by the readers which Lamport doesn't want to disclose is, perhaps, this [one](https://brooker.co.za/blog/2021/11/16/paxos.html).