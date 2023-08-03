---
title: TLA+&#58 A Design Tool For distributed System 
hide:
  - navigation
---

#### Introduction

Blueprint, Prose is not the right way
Dijstra: Formula
Lamport 40 year quote

<u>***Computation model:***</u> 
state: assignment of values to variables, a state is atomic wherein multiple variables are assigned values.
behavior: a sequence of state transitions
algorithm: set of all behavior

#### TLA+ & Distributed Systems
Concurrency, Failure, Non-deterministic message delivery.

#### Abstraction & Refinement
Pablo Picasso:
Nietzhe quote: 

#### Safety & Liveness
A program/algorithm execution can be viewed as sequence of states with values being assigned to variables in each state, conditions expressed on such executions(state transitions) for validity are called properties. Such properties validating an algorithm fall under two categories viz. safety & liveness. Large program are expressed as satisfying a set of safety/liveness properties.

<u>Safety:</u>
Safety is defined as "bad things shouldn't happen during the execution". If a bad thing does happen then: 
* It is irremediable & can identified at a certain point during the program execution i.e it is **discrete**. 
* Hence safety properties are supposed to hold **always** i.e every state transition.
 
Mutual exclusion, deadlock are examples where safety property is violated.
Formally, an execution is said to satisfy safety properties if every prefix of sequence of state transitions satisfies the safety properties. 

Safety conditions alone are not sufficient: 
A system can be safe/correct without doing anything but then it is useless.

<u>Liveness:</u>
Liveness property are complementary to safety property.Liveness is defined as "Good(useful) thing happens during the execution".Liveness property is something which will be satisfied eventually hence it is **non-discrete**(unlike safety property). It is something on lines of quote by Cicero: *"While there is life, there is hope."*

Livelock, starvation are examples where liveness property is violated.


mutual exclusion:
* safety: no two processes can be executing the critical section at same time.
* liveness: eventually, every process gets an opportunity to execute the critical section. 

Formally, every finite sequence of state transition can be suffixed with more state transitions with the possibility of satisfying liveness property. 

<u>Safety & Liveness:</u>

It is to be noted that any property can be expressed as conjunction of safety & liveness properties. 
Informally, an algorithm should remain correct along the course of its execution(safety) & produce useful results eventually(liveness) without getting stuck/looping forever etc.

(Mathematics is precise)
* Precisely express safety & liveness conditions. 
* Vet the solution using the safety 
* All possible executions are tested - algroithm's state space is explored.

#### Logic of Actions

#### Temporal Operators

Safety & Liveness using TLA+.

#### The plus in TLA+

#### Stuttering & Fairness

#### TLC

#### TLA+ Resources
Ron Presler: video
https://perspectives.mvdirona.com/2014/07/challenges-in-designing-at-scale-formal-methods-in-building-robust-distributed-systems/