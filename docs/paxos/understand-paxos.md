---
title: "Understanding Paxos With Three Simple Rules"
hide:
  - navigation
---

### **Introduction**
Consensus is defined as a group of processes(host machines) agreeing on a single value or request.
Such a group of processes provide redundancy using which fault tolerant systems can be built.

For building practical fault tolerant systems the group of processes need to agree on a series of requests while maintaining the
relative order of the requests. Such a group of processes can be used to implement a Replicated State Machine[^1]<sup>,</sup>[^2](RSM).
We will not discuss about RSM in this post, only single value agreement called single decree paxos will be discussed.

Consensus algorithm should meet the following conditions[^3]:
!!! note narrow ""
    * **Agreement:** A single value is chosen.
    * **Validity:** The value agreed upon value is actually proposed. This is to avoid trivial solutions,
    all processes can simply decide on a fixed value, say 42, no matter what.
    * **Termination:** Eventually a value gets chosen.

### **Environment**
An algorithm needs to consider the environment in which it is supposed to work. Following are the assumptions w.r.t 
processes & the network that make up the system. Processes communicate with each other by sending messages over the network.

***Process:***

* Processes run at arbitrary speeds, hence a process can take arbitrary time to respond back to a message.
* Processes may crash & restart. It is assumed that processes are equipped with stable storage to remember the chosen value across a crash. 

***Network:***

* Messages can get arbitrarily delayed in the network.
* Messages can get duplicated, reordered or lost.

Since the processes & network don't provide any guarantee w.r.t time, the system is said to be **asynchronous**.
In an asynchronous system, the algorithm should not make any assumption w.r.t time(*bounded message delay or processing time*) 
i.e typical mechanisms like **timeout** can't be used.

The net effect of above assumptions w.r.t timinig is that, in an asynchronous system, it is not possible to determine a process failure.
Suppose process **P1** sends a message to another process **P2**. The reply from the process **P2** can't be expected in bounded time either because the 
reply gets delayed in the network or the process might be acting slow. Hence failure detection is not possible as failure to receive response 
can't be attributed to process failure. The network/process delay can be infinite, theoretically.

!!! note narrow ""
    <center>***In an aysnchronous system, deterministic failure detection of a process is impossible due to message/process delay***

Note that the timing assumptions are fundamental while persistent storage & message loss, duplication & reordering are engineering challenges.
Message loss can be handled through retry(which can lead to duplication), message duplication & reordering can be handled by numbering the messages.

### **System**
The system (depicted in the picture below) consists of proposers & acceptors. 
Proposers receive requests from external clients & propose the requests to the acceptors.
Proposers don't hold any system state, it's the acceptors which hold the system state i.e the accepted value.
The algorithm doesn't require proposers or acceptors to communicate among themselves inorder to achieve consensus.
It is possible for a single process to act as both an acceptor & a proposer, for simplicity acceptors & proposers will be considered to be distinct.

!!! note inline left ""
    ```
    client requests   P: Proposer
          |           A: Acceptor
        --------      R: Request
        |      |
        R1     R2
        |      |
        \/     \/                            
    +----------------+
    | P1 P2 P3 ..    |
    | A1 A2 A3 ...An |
    +----------------+
    ```

!!! question "Why are multiple proposers required?"
    *Multiple acceptors are required to provide redundancy but why do we need more than one proposer to be present at the same time?
    Since a proposer is only required to propose a value & doesn't hold any system state, it should be possible to replace it in case of failure.*   

    While a proposer can be replaced **suspecting** failure, it is not possible to tell whether the proposer is dead as determinisitic process failure 
    detection is impossible in an asynchronous system. Hence it becomes imperative to assume the presence of more than one proposer at the same time.

### **Safety & Liveness**
Leslie Lamport introduced the concept of safety & liveness properties to define the **correctness**[^4](i.e behave as expected) of an algorithm.
Informally(proved formally[^5]), they are defined as:

* Safety: Something bad never happens.
* Liveness: Something good will eventually happen.

They are conditions expressed on the execution of an algorithm. 
Safety is the condition to prevent something bad from ever happening 
while liveness is the condition to express something good will happen eventually.
Safety condition is expected to hold true always, ensuring every step of an algorithm's execution is correct. 
Liveness condition holds true eventually & hence it can be thought of as the condition to express the end result. 

For instance, safety & liveness properties for mutex are given below:

* Safety: No two processes should be executing the critical section at the same time. 
* Liveness: Every process should eventually get a chance to execute the critical section.

With respect to consensus, **Agreement & Validity** are safety properties and **Termination** is liveness property.
Paxos has some issue w.r.t liveness owing to FLP[^6] result which will be discussed later.

### **Consensus**

When do we say consensus is achieved? It can't be defined as all acceptors accepting a single value. By definition, the algorithm is required to provide fault 
tolerance by sustaining some of the acceptor failures.

Assume the system consists **2N+1** acceptors. Any group of atleast **N+1** acceptors forms a majority & is called as **quorum**.
Any two quorums have atleast one acceptor in common. This property is called quorum intersection property.

If acceptors are allowed to accept only once, then acceptance of a single value by majority is made possible due to quorum intersection property.
Thus consensus can be defined as acceptance of single value by a majority of the acceptors.
Also, it should be possible to sustain upto **N** acceptor failures as acceptance of same value by a majority of acceptors is sufficient for consensus.

If a proposer receives response from atleast a majority of the acceptors, then it can conclude consensus is achieved.
Other proposers can determine whether consensus is achieved by querying the acceptors to see if a majority of the acceptors have accepted same value. 
Since consensus is defined as majority acceptance, the number of acceptors need to be ***fixed & known apriori*** to the proposers.

Consensus needs to be achieved amidst concurrent proposals by multiple proposers while sustaining proposer & acceptor failures. 
How concurrency & failure affect consensus will be discussed in the next section.

!!! note narrow ""
    <center>***Consensus is said to be acheived when majority of the acceptors accept same value. 
    <br>Consensus is concerned with two problems viz. <u>concurrency & failure</u>.***

### **Rule-One: Proposers Retry**

It might not be possible to achieve consensus under certain circumstances. Consider the following cases:

* **Case-1:** **N** acceptors accept value **v1**, and another **N** acceptors accept value **v2**. If one acceptor dies then consensus is not possible. 
In general, different group of acceptors can accept different values with non of the groups being a majority. Note that, in this case, consensus is not achieved even without any failure.

* **Case-2:** **N+1** acceptors accept value **v1**(i.e successfully persist the value) & another group of **N** acceptor accept value **v2**. But one or more acceptors crash before sending the response to the proposer. In this case, it creates ambiguity in determining whether consensus was achieved. If the failed acceptors come back online then
proposers will be able to see that consensus is achieved.

In both the cases it's a dead end, consensus was not possible due to concurrency(case-1) or failure(case-2) even with less than **N** failures.
To circumvent the above problem two changes are needed to the algorithm viz. acceptors are allowed to accept more than one value & proposers retry.
Hence more than one attempt may be required to achieve consensus.

!!! Note narrow "***Rule-1: Proposers Retry***"
    ***<center>Due to concurrency & failure, proposers might not be able to achieve consensus in a single attempt. <br>Hence proposers need to retry to achieve consensus.***

Now there can be multiple proposals proposing different values which are executing concurrently.
We will assume that the proposals are **totally ordered**, how this can be achieved will be discussed in the section [paxos protocol](#paxos-protocol).
Each proposal is assigned a proposal number using which the proposals will be ordered. The ones which start later have a higher proposal number.
A value is proposed using a proposal number, the acceptors store value & the corresponding proposal number. Also, it's assumed that a single value is proposed per proposal number.

With proposals being ordered, whenever a new proposal is started there are two possibilities:

* **Rule-2:** Prevent a prior ongoing proposal(having lesser proposal number) from achieving consensus.
  <br>If all prior proposals are prevented from achieving consensus then any value can be proposed in the current proposal.
* **Rule-3:** If above is not possible, then proposer has to choose a value from one of the ongoing proposals for the current proposal.
  A proposer might be behaving slow & could complete the proposal leading to consensus or it could be the case that one or more acceptors died 
  just before sending a response to the proposer as discussed earlier.

The protocol works in two phases viz. prepare & accept.

* **Prepare:** In the prepare phase a proposer gathers the current system state(values accepted by acceptors) by querying atleast a majority of the acceptors. 
*Rule-2* is applied at the acceptors during prepare phase, while *Rule-3* is applied at the proposer after the completion of prepare phase. 
For every new proposal both the rules are applied.
* **Accept:** Based on the system state gathered in prepare phase, a proposer chooses a value(ensuring safety) & proposes it to the acceptors in accept phase.
Note that the accept phase can be started only after completion of prepare phase.

### **Rule-Two: Kill The Past**

Consider the situation in the below diagram. **P1** is an ongoing proposal & a new proposal **P2** is started. 
**P2**'s proposal number is greater than **P1**'s proposal number.

``` title="An acceptor common between two proposals hasn't accepted any value"

    |<---P1--->|            * N1 & N2: Process groups consisting of N acceptors each.
    +------+---+------+     * P1 & P2 have an acceptor "A" in common, P2.proposal_number > P1.proposal_number.
    |  N1  | A |  N2  |     * "A" hasn't accepted a value from P1.
    +------+---+------+     * N1 & N2 don't have any other acceptor in common.
           |<---P2--->|    
```

1. Acceptor **"A"** which is common between proposals **P1** & **P2** hasn't accepted any value.
2. Since acceptors in **N1** are not involved in **P2**, it is not possible for proposal **P2** to know what value 
acceptors in **N1** have accepted(or will accept in the **future & achieve consenus**).
3. **P2** can choose to propose any value, however it should ensure **P1** doesn't achieve consensus(potentially for a different value).
4. To deal with the above situation, during the prepare phase, proposer for **P2** extracts a promise from acceptors in **N2** & **"A"** 
not to accept any proposal lesser than itself.
5. Hence proposal **P1** will not be able to achieve acceptance from a majority i.e **P2** just killed **P1**.
   Note that a quorum of acceptors (**N2** + **A**) have moved to a higher numbered proposal hence **P1** will not be able to achieve consensus.

In general, there could be more than one proposal before the current proposal, all those proposals in which *<u>at least one common acceptor</u>* hasn't yet accepted a value will be killed by current proposal. Also, there may be more than one acceptor common between any two proposals.

!!! note narrow "Rule Two: Kill The Past"
    ***<center>During the prepare phase, extract a promise from a quorum of acceptors participating in the current proposal <br>not to accept any proposal having a proposal number lesser than itself***

### **Rule-Three: Past Is Prologue**

Consider the situation in the below diagram. **P1** is an ongoing proposal & a new proposal **P2** is started.

=== "Single Accepted Value From Previous Proposal"
      <center>**Note: Make sure you read next tab after reading this one.**</center>
      ```title="Acceptor A which is common between proposals P1 & P2 has accepted a value."

      |<---P1--->|            * N1 & N2: Process groups consisting of N acceptors each.
      +------+---+------+     * P1 & P2 have an acceptor "A" in common, P2.proposal_number > P1.proposal_number.
      |  N1  | A |  N2  |     * "A" has already accepted a value from P1, A.value = P1.value.
      +------+---+------+     * N1 & N2 don't have any other acceptor in common.
             |<---P2--->|
      ```

      1. An acceptor **"A"** has accepted **"a"** as a value from **P1**.
      2. If a different value, say **"b"**, were to be chosen in proposal **P2** then: 
           * Acceptors in **N1** can complete the proposal **P1** which chooses **"a"**. 
             Note that proposal **P2** doesn't has any control over the acceptors in **N1** as they are not part of the proposal.
             It's also possible **P1** has already reached consensus.
           * Proposer for **P1** would deem **"a"** as chosen value & proposer for **P2** would deem **"b"** as chosen value.
           * Ultimately, though a majority has accepted **"b"** after **P2** completes, there would have been a <u>flip from **"a"** to **"b"**</u> 
             at some point in time if proposal **P1** completes before **P2**. 
      3. To avoid above situation, before proposing a value in proposal **P2**, the proposer tries to learn if any of the acceptors in **P2** had already 
         accepted a value from past proposal during the prepare phase. If so, then the previously accepted value is used for the current proposal **P2**.
      4. We are still not done, what if more than one common acceptors have accepted different values from different past proposals.
         A question that arises is **which value** should be considered for the current proposal? Continue to the next tab for answer.
         
=== "Multiple Accepted Values From Different Past Proposals"
      <center>**Note: Make sure you have read previous tab before reading this one.**</center>

      Given a past proposal & current proposal, the past proposals will fall into two categories:

      * At least one common acceptor hasn't accepted a value, such proposals will be killed following **Rule-2**.      
      * One or more common acceptors have accepted a value from past proposals. The past proposals will be in one of these states: ***dead, ongoing or consensus achieved***. 
        However, the current proposal doesn't know about the <u>status of the past proposals</u>, a proposer might be behaving slow & could complete the prosposal leading to consensus.
        Current proposal has to choose one value from the different values proposed in these past proposals ensuring ***safety***.
        <br>**Note:** Dead proposal means consensus can't be achieved for that proposal.
      
      Lets start from the beginning. Considering the very first two proposals, two things are possible: 

      * **P2** proposal kills **P1**, the value proposed in **P2** & **P1** proposals can differ(following Rule-2). <br>We get the sequence: =="dead-ongoing"==.
      * **P2** chooses the same value as **P1**(as discussed in previous tab), both **P2** & **P1** are ongoing. <br>We get the sequence: =="ongoing-ongoing"==.

      For the third proposal, the following below given scenarios are possible in the prepare phase when a query is made.
      ```
          Scenario --> a               b               c               d
                       ------------------------------------------------------------    
          Proposal --> dead-ongoing    dead-ongoing    dead-ongoing    dead-ongoing
          Status          ^       ^       ^                       ^ 

        ^: Indicates one or more acceptors from this proposal respond with the value accepted in this propsoal  
           when a prepare request is made for current proposal.

        a: The accepted values correspond to dead & ongoing proposal. 
           Here value corresponding to ongoing proposal needs to be considered for current proposal 
           to preserve safety.
        
        b: The accepted value corresponds to dead proposal. Since no acceptor participating in the current proposal 
           has accepted a value from the ongoing proposal, the ongoing proposal will be killed.
           So it is safe to consider the value corresponding to the dead proposal.
        
        c: The accepted value corresponds to an ongoing proposal. 
           It's safe to choose this value for current proposal.
        
        d: None of the accceptors in current proposal have accepted a value in past proposals. 
           All the past proposals(including already dead, aha) will get killed.
           The current proposal is free to choose any value from the wild for the current proposal. 
      ```
      When a new proposal **P3** starts, it is possible to get a sequence like: =="ongoing-dead"== i.e **P2** gets killed while **P1** is active.
      Though such a sequence is possible it is still equivalent to =="ongoing-ongoing"==, as the dead proposal will have the same value as the ongoing proposals(for safety we are concerned only with the value). In general, a sequence like =="ooooddd"== or =="oododod"== are equivalent to =="ooooooo"==. Generalizing, it is easy to see that we need to consider a sequence like: =="ddddooooo"==, dead proposals followed by ongoing proposals.
      
      Given a sequence like =="ddddooooo"== & if more than one acceptor in current proposal had already accepted a value in past proposals, then the value corresponding to the 
      **highest numbered past proposal** needs to considered for current proposal. If the highest numbered proposal corresponds to an ongoing proposal, the sequence gets extended with another ongoing proposal. However, if highest numbered proposal corresponds to a dead proposal then all the ongoing proposals will get killed & the new sequence will have only one ongoing proposal.

      **Another perspective:** Assume value corresponding to highest numbered past proposal is considered for the current proposal, any prior proposal would have done the same thing. 
      In other words Paxos is a recursive algorithm, the above explanation is based on unrolling the recursive execution of Paxos algorithm.

      Once consensus is achieved, only that value can be used in all future proposals as all future proposals will have at-least one common acceptor with
      the proposal which achieved consensus due to quorum intersection propoerty(note, any higher numbered proposal will also have same value). This holds true even if there are upto **N** failures. This corresponds to **Case-2** that was discussed under section **Rule-1**.

      Also, note that acceptors need not have to remember all the values proposed in the past proposals, the one corresponding to the highest numbered proposal is sufficient.
      
!!! note narrow "Rule Three: Past Is Prologue"
    ***<center>If acceptors participating in the current proposal had already accepted a value from different past proposals 
    then, <br>for the current proposal use the value corresponding to the highest-numbered past proposal.***


### **Paxos Protocol**
**Choosing proposal number:**
Whenever a new proposal is started, the below two conditions should be met w.r.t proposal number: 

* A single value is proposed per proposal number.
* The proposal number should be greater than all the past proposals. 

The following mechanism can be used to meet the above conditions.<br> 
Proposers should choose a proposal number from disjoint sets & a proposer should not reuse a proposal number to propose a different value(this can be achieved if a proposer persists it's last used proposal number & doesn't reuse it again). 

Proposal number can be generated as a concatenation of two numbers: <***I***,***J***>, ***J*** corresponds to the proposer number(facilitates disjoint sets) & ***I*** is a monotonically increasing number. At the start of a new proposal, a proposer queries atleast a majroity of the acceptors to discover the highest ***I*** value & then increments it to generate it's proposal number.

If a proposer chooses a proposal number as above, then it need not store it's last proposal number locally on stable storage to avoid resuse of the proposal number.
This is made possible because a prepare request is made(atleast to a majority) before an accept request & value comes into picture only at the accept phase. 

The complete paxos algorithm is summarized below[^3], phase-1 corresponds to prepare & phase-2 corresponds to accept:

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
        [The Bug in Paxos Made Simple](https://brooker.co.za/blog/2021/11/16/paxos.html).

### **Discussion**

#### Moment of consensus
When do we say consensus is achieved? It's at the moment the last acceptor in a quorum successfully persists the accepted value.
Interestingly, neither the proposer nor any of the acceptors know about this celebratory moment.
Both the proposer & the acceptor may fail the very next moment. 
However, from this moment onwards, for any new proposal only **Rule-3** will apply & consensus can only be achieved for the same value. 

Note that an acceptor need to persist proposal number(during prepare phase) & accepted value(during accept phase) before replying to the
the proposer to ensure safety.

#### Time & Order

How do we establish order(i.e one occurred before the other) between events happening at different places(i.e processes) in a distributed system without a global reference like clock.
Lamport has addressed this problem in a paper[^7]. To establish order between two events in a distributed system either of the below given two conditions is required:

* Events happen on the same process. 
* A message is sent from one process to another. 

Here we want to establish order between proposals that are all started on different proposers.
When a prepare request is made by a proposer the proposal information(i.e proposal number) is made available on a quorum of acceptors. When a proposer wants to start new proposal a query is made to atleast a quorum of acceptors inorder to choose a proposal number. Thus these two events **serialize** on the common acceptors in the quorum(i.e *<u>the first condition mentioned above</u>*). This fact is leveraged by the mechanism explained in the [paxos protocol](#paxos-protocol) while choosing a proposal number. Hence proposal numbers can be thought of as Lamport timestamps.

Note that, proposal numbers need not be lamport timestamps. Proposers may choose any random number, **Rule-2** simply disallows lesser numbered proposals.
On a sidenote, since we are talking about space(different processes) & time(order of events), one can look at "moment of consensus" as occurring at a particular <u>***place & time***</u>.

#### Safety & Liveness

**Safety:**
Earlier we stated agreement as safety property for Paxos.
However that's still a very high level requirement, safety property needs to be defined considering multiple concurrent proposals.
Given multiple concurrent proposals below given conditions guarantee the agreement condition i.e single value is chosen.

* If a value **v** is being chosen for proposal P then no value other than **v** can be chosen for any proposal Q < P.
  This translates to below conditions:

    * No value is chosen in any propsal Q < P i.e proposal will not complete, rule-2 is used to meet this condition.
    * Value **v** is already chosen or can be chosen(proposal being in progress), rule-3 is used to meet this condition.
    
    In general, for any new proposal both the rules are applied i.e some proposals get killed by applying **rule-2**, while a value for current proposal 
    gets decided as per **rule-3**.

* A single a value is proposed per proposal number.
  Though this condition does not lead to safety violation if it's not satisfied, it's required to avoid confusion.
  <br>For safety violation a different value needs to be chosen. Let's say an acceptor **A** has accepted value **x**, and **N+1** acceptors accepted 
  value **y** for same proposal number. Assume acceptor **A** is part of a quorum for any new proposal, atleast one acceptor from the remaining **N+1** is required for current proposal
  to form a quorum. This only leads to confusion w.r.t the value to used for current proposal.

**Liveness:**
**Rule-2** causes liveness issue. Consider the below diagram in which two proposers keep killing each other's proposals forever.
A proposer **P1** starts a proposal & before it can start accept phase another proposer **P2** starts a new proposal.
And this process continues forever & consensus will never be achieved. 
For simplicity, the common acceptor is shown to be the same throughout.
Note that the scenario can be generalized to more than two proposers with newer proposals not allowing older proposals to compelete & 
new proposals keep getting started forever. This issue is due to the famous FLP[^6] result.

```title="Paxos liveness issue"

P1 ---<1>---------[1,a]----<3>---------------[3,a]----      <n>: proposal request
        \           \       \                  \            [<proposal-no>, <value>]: accept request  
         \           \       \                  \           x: rejected
A1 ------<1>---<2>----x------<3>----x----<4>-----x---
               /                   /      / 
              /                   /      / 
P2 ---------<2>----------------[2,b]----<4>----------
```

However this is not a major issue in practice. 
Consensus terminates if there is single leader for sufficient time so that it can complete both the phases viz. prepare & accept.
The issue can be avoided in practice by using techniques like randomization.

!!! note narrow ""
    <center>***Safety & Liveness properties are better way to reason about concurrent algorithms as number of possible executions<br>for a concurrent algorithm could be on astronomical scale.***

For more details go through the following presentations from Leslie Lamport:

  * Voting Algorithm:  [Leslie Lamport — The Paxos algorithm or how to win a Turing Award. Part 1.](https://www.youtube.com/watch?v=tw3gsBms-f8)
  * Paxos Algorithm: [Leslie Lamport — The Paxos algorithm or how to win a Turing Award. Part 2.](https://www.youtube.com/watch?v=8-Bc5Lqgx_c)  

These presentations are highly recommended. A key take away from the above presentation is about how Leslie Lamport uses [abstraction](https://youtu.be/pgWTmOyUjtM?t=517) (i.e remove irrelevant detail). Voting algorithm is presented as precursor to Paxos, messaging is considered as a detail & proposers are abstracted out. 
Acceptors need information to make decisions, how this information is made available is immaterial, instead what an acceptor does with the available information is the main concern. 
What Leslie Lamport refers to as ballot in the above videos is actually proposal number. The presentation uses TLA+[^], Paxos algorithm is formally proven using TLA+.  

#### Asynchrony

Asynchrony assumption leads to failure detection impossibility which in turn leads to multiple proposers. 
With multiple proposers concurrency comes into picture & concurrency means complexity. 
That begs us to ask the question - are the assumptions too stringent which results in algorithm complexity.
Do the timing assumptions really matter in practice or are they merely of theoretical interest. Are we being pragmatic or a purist?

One might be tempted to think, for practical applications timeout could still be a viable option i.e after a certain time period
a process can be considered to have failed. However it is not that simple. 
If you choose the timeout too low, safety is affected as the suspected process might come back & start participating in the 
algorithm creating adverse affects, choose it too large system availability is affected. 
Also, timeout can't be chosen statically, it depends on the network conditions & needs to be tuned dynamically for optimal performance.

Since Paxos is used in building very fundamental primitives like distributed lock safety guarantees are of utmost importance.
Fundamental primitives like lock can't be unreliable. A compromise in safety means adverse affects on the whole system 
& recovering it may lead to long outages or could even result in irrecoverable damages like data loss.

Some stories from the trenchies are discussed in this paper: [The network is reliable](https://dl.acm.org/doi/pdf/10.1145/2643130).
At GitHub, a 90 second network delay due to network partition caused processes killing each other by issuing STONITH messages.
This is exactly what we are discussing about w.r.t timing assumptions. 

!!! quote "A github story"
    This 90-second network partition caused fileservers using Pacemaker and DRBD for HA failover to declare each other dead, and to issue STONITH (Shoot The Other Node In The Head) messages to one another. The network partition delayed delivery of those messages, causing some fileserver pairs to believe they were both active. When the network recovered, both nodes shot each other at the same time. With both nodes dead, files belonging to the pair were unavailable.

If an algorithm is affected by external environment factors like time then the algorithm's behaviour could be unpredictable. 
In another instance leap second affected linux mutex[^8]<sup>,</sup>[^9] which affected systems globally. It's best if time can be taken out of the equation. 
It's good that consensus does not need to depend on time but that's the not case for a mutex, if you are interested 
**ref:** *section 3.3: Mutual Exclusion versus Producer-Consumer Synchronization*(Concurrency, The works of Leslie Lamport).

Further discussion about the topic can be found in the book Replication Theory, *Chapter 4: Stumbling over Consensus Research: Misunderstandings and Issues* & 
the paper *Consensus: The big misunderstanding*[^10]

There has been some work on consensus considering partial synchroncy[^11]<sup>,</sup>[^12].

On a independent note, for further exploration refer to the history[^13] & paxos variants[^14].

[^1]: Implementing Fault-Tolerant Services Using the State Machine Approach: https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf
[^2]: Building Highly Available Systems: https://www.microsoft.com/en-us/research/publication/how-to-build-a-highly-available-system-using-consensus
[^3]: Paxos Made Simple: https://lamport.azurewebsites.net/pubs/paxos-simple.pdf
[^4]: Proving the Correctness of Multiprocess Programs: https://lamport.azurewebsites.net/pubs/proving.pdf 
[^5]: Defining Liveness: https://cs.nyu.edu/~apanda/classes/fa22/papers/alpern85defining.pdf
[^6]: Impossibility of Distributed Consensus with One Faulty Process: https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf
[^7]: Time, Clocks, and the Ordering of Events in a Distributed System: https://lamport.azurewebsites.net/pubs/time-clocks.pdf
[^8]: Leap Second crashes half the internet: https://www.somebits.com/weblog/tech/bad/leap-second-2012.html
[^9]: The Inside Story of the Extra Second That Crashed the Web: https://www.wired.com/2012/07/leap-second-glitch-explained/
[^10]: Consensus: The big misunderstanding: https://web.cecs.pdx.edu/~greenwd/guerraoui97consensus.pdf
[^11]: On the Minimal Synchronism Needed for Distributed Consensus: https://dl.acm.org/doi/10.1145/7531.7533
[^12]: Consensus in the Presence of Partial Synchrony: https://dl.acm.org/doi/10.1145/42282.42283.
[^13]: A brief history of Consensus, 2PC & Transaction Commit: https://betathoughts.blogspot.com/2007/06
[^14]: Paxos Variants: https://paxos.systems/variants

