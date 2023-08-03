---
title: "Understanding Paxos With Three Simple Rules"
hide:
  - navigation
---

### Problem statement & definitions 
The problem of consensus is defined as a group of processes agreeing on a single value.

Formally, consensus needs to address the following three conditions:

* Agreement: All correct processes decide on the same value. 
* Validity: The value agreed upon is actually proposed(*this is to avoid trivial solutions, all processes can simply decide a fixed value no matter what, say, 42*).  
* Termination: Eventually all correct processes reach a decision. 

The first two are safety properties & last one is liveness property.

**Acceptors & proposers:**
The processes(*or hosts*) are divided into a set of acceptors & proposers(*ignoring learners*).
Proposers propose to acceptors in order to choose a value. It is possible for a single process to be an acceptor & proposer 
but for simplicity we will consider acceptors & proposers to be distinct. 

**Asynchronous system**: The processes are connected by a network & communicate with each other by exchanging messages.

* *Processes:* Processes can crash & restart, processes are equipped with stable storage. Processes can take arbitrary time to execute & respond back to a message. 
  The set of acceptor processes is fixed & known a priori. A process uses stable storage to remember critical information when it restarts after crashing.  

* *Network*: Messages can get arbitrarily delayed in the network. Messages can get duplicated, be delivered out-of-order & get lost. However the network 
  is considered to be reliable in the sense that if the sending process retries indefinitely then message will be delivered, eventually. 
   
* *Communication(timing)*: Since processes & network don't provide any guarantee w.r.t time no timing assumptions are made. 
  Processes can't use clock to accurately measure time interval or determine an epoch. 

!!! warning ""
    *A fundamental problem in an asynchronous distributed system is that it is impossible to tell ***whether a process has crashed or slowed down***.*
    <br>*Hence, it becomes imperative to ***neutralize a slow process's execution*** in order to proceed further so as to avoid adverse effects(ensure "safety").*
    ??? question "Why can't failure be detected using a ping"
        The expected time to receive a reply for a ping can't be bounded as, by definition, a process can take arbitrary time to reply back & 
        the time delay in network is unbounded.  

**Quorum:** Any sub set containing at least a majority of elements of a set is called a quorum. Given a set, any two quorums contain at least one element in common. 
<br>This key property is leveraged in ensuring safety property: ***single value is chosen, always***. 
<br>We will assume the system contains ***2N + 1*** acceptors, at least ***N+1*** acceptors are required to form a quorum.

**Choosing a value**: If a value is chosen by a quorum of processes, then chosen value can be learnt by querying a quorum of processes. 
Hence a group of ***2N + 1*** acceptors can sustain up to ***N*** acceptor failures. Thus the processes provide fault tolerance.   

!!! success "Choosing a value"
    *A value is considered to be chosen if it is **accepted by a quorum of processes**.*
    <br>*At <u>no point in time</u>, two different proposers should have different values accepted by a quorum of acceptors.*
 
### The Three Rules
#### <u>Rule One: Multiple Rounds</u>

Suppose acceptors are allowed to accept only once & there are two concurrent proposals i.e one for value ***a*** by proposer **P1** 
& another for value ***b*** by proposer **P2**. Now, assume ***N*** acceptors accept value ***a*** , another ***N*** acceptors accept value ***b*** 
& the remaining one has failed. Under this scenario there is no further progress possible unless more proposal attempts are made.
  
```title="Why can't we have a single proposer, always ?"
That single proposer can die or can become slow. 
If a proposer becomes slow then another proposer steps in, at this point the slow proposer 
can come back again. Thus we have two concurrent proposals.

Remember we can't distinguish between slow & crashed processes.
Note that there can be more than two concurrent proposals.
```

!!! note "Rule One: Multiple Rounds"
    ***<center><span style="color:blue">Due to concurrent proposers, more than one proposal may be needed for choosing a value. 
    <br><u>Safety:</u> Even with concurrent proposals a single value should be chosen, always.</span>***

***Note:*** Since multiple proposals are required, each proposal is numbered & the proposals are assumed to be totally ordered(*monotonically increasing*). 
A proposal is now expressed as: ==*< proposal-number, value >*==. In each proposal first a query is made to know the current status & then a value is
proposed for consensus.  

#### <u>Rule Two: Kill The Past</u> 
<todo query & set> <todo search host reference>
Consider the situation in the below diagram:
``` title="An acceptor common between two proposals hasn't accepted any value"

    |<---P1--->|            * Proposals P1 & P2 have an acceptor "A" in common, P2 > P1.
    +------+---+------+     * "A" hasn't accepted a value from P1, A.value = nil.  
    |  N1  | A |  N2  |     * N1 & N2 don't have anything in common.
    +------+---+------+
           |<---P2--->|    
```

1. Acceptor **"A"** which is common between proposals **P1** & **P2** hasn't accepted any value.
2. It is not possible for proposal **P2** to know what value acceptors in **"N1"** have accepted(or will accept in future).
3. **P2** can choose to propose any value, however it should ensure that a different value is not chosen by **P1**.
4. To deal with above situation, before proposing a value in proposal **P2**, proposer 
   for **P2** extracts a promise from acceptor **"A"** not to accept any proposal lesser than itself.
5. Hence proposal **P1** will not be able to achieve acceptance from a quorum i.e **P2** just killed **P1**.
6. There could be more than one proposal before **P2**, all those proposals in which *<u>at least one common acceptor</u>* hasn't yet accepted a value
   will be killed by **P2**.


!!! note "Rule Two: Kill The Past"
    ***<center><span style="color:blue;">Extract a promise from acceptors participating in the current proposal not to accept any proposal lesser than 
            the current one.***</span>

#### <u>Rule Three: Past Is Prologue</u>

=== "Single Accepted Value From Past Proposal"
      <center>**Note: Make sure you read next tab after reading this one.**</center>
      ```title="Acceptor A which is common between proposals P1 & P2 has accepted a value."

      |<---P1--->|            * Proposals P1 & P2 have an acceptor "A" in common, P2 > P1.
      +------+---+------+     * "A" has already accepted a value from P1, A.value = P1.value.
      |  N1  | A |  N2  |     * N1 & N2 don't have anything in common.
      +------+---+------+
             |<---P2--->|
      ```

      1. Acceptor **"A"** had already accepted **"a"** as a value from **P1** before **P2** started.
      2. If a different value, say **"b"**, were to be chosen in proposal **P2**. 
           * Acceptors in **"N1"** can complete the proposal **P1** which chooses **"a"**. 
             Note that proposal **P2** doesn't has any control over the acceptors in **"N1"**.
             It's also possible **P1** has already reached consensus.
           * Proposer for **P1** would deem **"a"** as chosen value & proposer for **P2** would deem **"b"** as chosen value.
           * Ultimately, though a quorum has accepted **"b"** after **P2** completes, there would have been a <u>flip from **"a"** to **"b"**</u> 
             at some point in time if proposal **P1** completes before **P2**. 
      *  To avoid above situation, before proposing a value in proposal **P2**, the proposer tries to learn if any of the acceptors in **P2** had already 
         accepted a value from past proposal. If so, then the previously accepted value is chosen for the current proposal **P2**.
      *  We are still not done, what if more than one acceptor has accepted different value from different past proposals. 
         A question that arises is **which value** should be considered for the current proposal? Continue to the next tab for answer.
      ----   

=== "Multiple Accepted Values From Different Past Proposals"
      <center>**Note: Make sure you have read previous tab.**</center>
      Whenever a new proposal starts it will able to know about all the past proposals by querying a quorum of acceptor. As per quorum
      property the current proposal will have one or more common acceptors in each of past proposals. The past proposals
      will fall into two categories, given a past proposal & current proposal: 

      * At least one common acceptor hasn't accepted a value, such proposals will be killed following **Rule-2**.      
      * One or more common acceptors has accepted a value, such proposals will be in one of these states: ***dead, ongoing or consensus achieved***. 
        However the current proposal can't know about the status of the past proposals all by itself(*It needs to be deduced through other past proposals*). 
        Current proposal has to choose one value from the different values proposed in these past proposals preserving ***safety***.
        
        !!! note 
            Safety requires that no two ongoing proposals choose different values.

      Consider the following sequence of possibilities, beginning with two proposals **P1** & **P2** 
      which have accepted values **aa**(both are ongoing i.e ***previous tab***) or **ab**(P2 has killed P1 i.e **rule-2**), respectively. 
      Subsequently value to be proposed by **P3** is deduced from past proposals **P1 & P2** ensuring safety. 
      In the process, in order to generalize, we arrive at the pattern of dead & ongoing proposals 
      which will be used for choosing a value to be proposed by any future proposal after **P3**.  

      <center><p><img src="/blog/paxos/paxos.png" alt="Multiple Accepted Values" width="800px"/></p></center>

      <u>**Choosing value for P3:**</u>

      * <u>Cases 7,3:</u> Both of these have two acceptors in current proposal which have accepted a value from different past proposals.
   
          * <u>case-7:</u> value corresponding to the higher numbered proposal i.e **"b"** should be used considering **safety**. Otherwise
                           two ongoing proposals will be proposing values **"a"** & **"b"**. 
          * <u>case-3:</u> either of them can be used, since we used higher numbered proposal w.r.t *case-7* same can be done in this case.
      
      * <u>Cases 4,8:</u>
            Acceptors in current proposal didn't accept a value in any of the past proposals.
            All past proposals get killed, current proposal is free to choose any value of it's choice. 
      
      * <u>Rest of the cases:</u> A single acceptor in current proposal accepted a value in a past proposal, choice is obvious.

      From here onwards, <u>highest-numbered proposal</u> refers to the largest-numbered past proposal from which an acceptor participating 
      in the current proposal had accepted a value in the past. And by <u>accepted values</u> we mean values accepted by acceptors common between 
      a past proposal & current proposal. 

      <u>**Choosing value for future proposals:**</u>

      * <u>cases 2 to 8:</u>The pattern observed is ongoing proposals follow dead proposals, e.g: ==**dddooo**==. 
      All the ongoing proposals will be choosing same value. 
      If two accepted values are considered then the pattern will be one of these: ==**dd**==, ==**do**==, ==**oo**==.<br>
      In this case as well highest numbered proposal needs to be considered for future proposal greater than **P3**.
      The same reasoning can be extended if there are more than two accepted values.
        
      * <u>case-1:</u> The dead proposal had same value as the ongoing proposals. If we consider
      first two as accepted values the pattern is ==**od**==, in this case as well higher numbered proposal can be used so that it **resolves
      uniformly** with all the other cases. In general, suppose the pattern is ==**ooooo**== & any proposal barring the first one gets killed, then above pattern results.

      <u>**Choosing value after consensus is achieved:**</u> 
      <br>Once consensus is achieved, one or more concurrent future proposals will choose consensus proposal 
      for highest-numbered proposal(*since a quorum of acceptors would have accepted a value*). For all future proposals, either 
      consensus proposal or one of those following it will be chosen for highest-numbered proposal. 
      *Cases 4,8* are not possible. Also, notice that up to **N** failures can be sustained.

    !!! note ""
        In all the cases, value corresponding to the ***highest-numbered proposal*** should be used for current proposal.

    ??? note "Paxos Recursion"
        
        If you remember, earlier it was said current proposal can't know the status(*dead,ongoing or consensus*) of past proposals all by itself. 
        By choosing value corresponding to the highest-numbered proposal it implicitly gets to know about the status of past proposals(*what's shown in
        the table above is base case*). 

        The recursive nature of Paxos is depicted below. While the value corresponding to the highest-numbered proposal gets chosen for current proposal, 
        those shown in grey boxes(*no common acceptor had accepted a value for proposals between highest-numbered proposal & current proposal*) will get 
        killed by the current proposal. The highest-numbered proposal would have done the same earlier(i.e deduce status), ensuring safety(*refer above table*). 

        !!! note ""
            ```mermaid
                graph LR
              
                A[dead]---B[highest]---C([dead])---D([ongoing])---E[highest]---F([ongoing])---G([ongoing])---H([dead])---I([ongoing])---J[current]
                
                J-->E-->B

                style F fill:#818589
                style G fill:#818589
                style H fill:#818589
                style I fill:#818589
            ``` 
    ----

!!! note "Rule Three: Past Is Prologue"
    ***<center><span style="color:blue;">If more than one acceptor participating in the current proposal had already accepted a value from different past proposals 
    then use the value corresponding to the highest-numbered past proposal for the current proposal to ensure safety</span>.***

### Paxos Algorithm
#### **<u>Two phase protocol</u>** 
To satisfy ***Rule-2 & Rule-3*** Paxos runs in two phases viz. ***prepare & accept***. 
Rules are applied during the **prepare** phase which is followed by **accept** phase to propose a value.      

??? info "Paxos Protocol"
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

#### **Safety & Liveness**
    
##### Safety
<u>***Concurrency:***</u>
Paxos uses a quorum whenever a new proposal starts which allows it to know about all past proposals.
Given a current proposal there are only two possibilities w.r.t past proposals. 

* at least one acceptor in the current proposal hasn't accepted any value in past proposals or 
* one or more acceptors in the current proposal have already accepted a value prior to current proposal 
  which can potentially complete along with current proposal. 
 
Accordingly, ***Rule-2 and Rule-3*** are applied to kill and choose a value for next proposal ensuring safety until a value gets chosen. 
Once a value is chosen only ***Rule-3*** gets applied which ensures consensus value is used for all future proposals.

***<u>Failure:</u>***
Paxos is resilient to proposer failures as it is safe for concurrent proposals.
Acceptor failures are sustained due to quorum technique. 

<u>***Totally ordered, unique proposal numbers:***</u>
Paxos requires proposal numbers to be total ordered & unique. 

This can be ensured if all proposers choose the proposal numbers from disjoint sets which are monotonically increasing. 
For instance, this can be done by having the proposal number to be concatenation of a monotonically 
increasing integer number & acceptor number(acceptors are numbered 1 through n). 

The first part of the proposal number can be constructed by contacting a quorum of acceptors to know about highest numbered proposal 
& incrementing the first part of the proposal for next proposal.

<u>***Out-of-order message delivery:***</u>
Since accept phase is started after prepare phase completes out-of-order message delivery is safe.
If quorum used in prepare & accept differ then some of the hosts will receive accept message without receiving any prepare request.
In such a case acceptors should [update the proposal number](https://brooker.co.za/blog/2021/11/16/paxos.html) as well(*which is done in prepare phase*).

<u>***stable storage:***</u> Paxos requires acceptors to remember the accepted value if an acceptor crashes & restarts. An acceptor 
stores the accepted value on stable storage before replying back to the proposers.  

##### Liveness
Note that initially we stated that eventually a value should be decided by all correct processes. 
Under certain condition this is not possible, this is not a serious limitation though & can be circumvented for practical purposes.

**Rule-2** causes liveness issue, lets say an ongoing proposal is killed & new one starts. But then the current proposal can get killed by a future proposal 
& the process can continue endlessly. Hence a value never gets chosen. The scenario is illustrated in the diagram below.
There are two proposers & for simplicity a single acceptor(common between the quorums) is being shown. It keeps on rejecting 
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

Note that while arriving at ***Rule-1*** the necessity for multiple proposers was introduced in the context of faulty proposer(crashed or slow). 
Paxos simply assumes multiple proposers & hence the above issue. The solution it suggests is to have a single distinguished proposer called leader.
The liveness issue doesn't vanish but it is cleverly diverted to leader election(where it prevails). 
If the leader is given sufficient time to complete both the phases & the system(proposer, acceptor & network) is working properly 
then consensus is achieved.

??? "FLP in the context of Paxos" 
    Liveness issue discussed above is attributed to the famous FLP proof which asserts that consensus is **impossible even with a single faulty 
    process** in an asynchronous system. What FLP really means is that consensus is not possible **sometimes** under certain conditions. 
    Consider the below statements taken from the FLP paper.

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

    In practice this is not a major problem and the issue can be avoided by using a reasonable timeout to give enough time to complete
    the execution or by using randomization to avoid getting into a dueling situation(illustrated earlier using two proposers stepping on each
    other's execution).

### Multi-Paxos

Practical systems require that a series of values be decided instead of just one value. The decided values are added to a log 
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

For realizing a replicated log multiple instances of paxos can be run as shown in the above diagram on the left side with value
in the proposal now consisting of: ***< index, value >***. However this is not an efficient implementation. In a more practical implementation 
a single proposer(called leader) can issue just accept requests for multiple slots as shown in the above diagram on the right side. 
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

### What's Next 
**Basic Paxos:**
Leslie Lamport first developed Paxos as voting algorithm which uses shared memory model. If you are still not clear with above explanation you can check 
out the voting algorithm here (you will get to learn TLA+ as well):

* [The Paxos algorithm or how to win a Turing Award. Part 1.](https://www.youtube.com/watch?v=tw3gsBms-f8)
* [The Paxos algorithm or how to win a Turing Award. Part 2.](https://www.youtube.com/watch?v=8-Bc5Lqgx_c)

You can also check out: 
Lampson's [How to Build a Highly Available System Using Consensus](https://www.microsoft.com/en-us/research/uploads/prod/1996/10/Acrobat-58-Copy.pdf).
& [Paxos lecture (Raft user study)](https://www.youtube.com/watch?v=JEpsBg0AO6o)

**Multi-Paxos & Reconfiguration:**
Two things complicate Paxos: need to choose multiple values for practical applications & change the acceptors in the system.

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
<br>[Corfu](https://www.usenix.org/conference/nsdi12/technical-sessions/presentation/balakrishnan) talks about implementing various distributed systems 
like transactional KV store, databases etc & AuroraDB views [log as the database](https://assets.amazon.science/dc/2b/4ef2b89649f9a393d37d3e042f4e/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases.pdf). 

Hence Paxos is a very fundamental algorithm for distributed systems. 

**Variants:** For different variants of paxos you can checkout: https://paxos.systems/variants/

**Trivia:**
Lastly, it would be interesting to read about the initial reception about [paxos](https://lamport.azurewebsites.net/pubs/pubs.html#lamport-paxos) & a subtle 
bug referred to [here](https://lamport.azurewebsites.net/pubs/pubs.html#paxos-simple) due to interpretation by the readers which Lamport doesn't want to disclose 
is, perhaps, this [one](https://brooker.co.za/blog/2021/11/16/paxos.html).
