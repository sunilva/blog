---
title: TLA<sup>+</sup> Spec for View Stamp Replication
hide:
  - navigation
---

#### Introduction
paper: https://pmg.csail.mit.edu/papers/vr-revisited.pdf

View Stamp Replication is a replication algorithm for realizing replicated state machines 
which is the basis for building fault tolerant distributed systems..

You can read about the analysis of the algorithm in my earlier [post](analyze-vsr.md) which concluded that TLA+ analysis is imperative.

Distributed systems have three problems viz. ***concurrency, failure & network behavior*** as far as algorithm design is concerned. 
TLA+ is an ideal tool for modeling concurrent systems & TLA+ provides some constructs to model 
network behavior(specifically, out-of-order delivery which also means delay, implicitly).

Failure & message loss/duplication need to be implemented in the spec, explicitly.

Note: Failure is fairly easy to reason about on paper as compared to concurrency & non-deterministic message delivery as 
innumerable different executions are possible.

#### Implementation
Please go through the post mentioned above before proceeding further, at least 
go through the sections: [solution](analyze-vsr.md#solution), [abstract representation](analyze-vsr.md#abstract-representation) & [analysis](analyze-vsr.md#analysis).

Basically the system consists of replicas exchanging messages, so the spec is defined using: 

* replicaState: TLA+ function to represent replicas. A couple of helper functions named `update*` are provided to manipulate replicaState.
                Generally, replicas are represented using multiple TLA+ constructs which compromises on encapsulation.
* messages: TLA+ record to represent the messages passed between replicas.

The spec doesn't model client & state machine interactions in order to keep the model size small.
Also, the model considers only log based implementation i.e application state is ignored.

Currently, the spec implements Normal & View-Change sub-protocols only. 
<br>Spec: https://github.com/sunilva/tlaplus-specs/blob/main/ViewStampReplication.tla

!!! warning "Limitations"
    
    * GET-STATE is not implemented. 
    * Message loss(network partition) is not implemented.

#### Testing

The spec has been verified with following model configuration:

|request  | replicaCount  |viewChangeTriggerLimit | state space size |
|---------|---------------|-----------------------|------------------|
| {1,2,3} | 3             | 0                     | 3M               |
| {1,2}   | 3             | 1                     | 1M               |
| {1,2,3} | 3             | 1                     | 205M             |
| {1,2}   | 3             | 2                     | 44M              |                            

**Notes:**

 * NIL is a model value.
 * Uncheck deadlock from the model definition.

!!! tip
     * Run the model without checking any safety & liveness properties, this will give an idea about 
        the size of the state space. 
     * Once you know state space size check for safety properties followed by liveness properties.  
     * Checkout https://vimeo.com/264959035 for additional tips.

#### Results

* Concurrent executions are hard to reason about on paper. TLA+ proves to be very useful given that there will be innumerable possible executions. 
  TLA+ spec serves as a ***completely tested prototype*** since every possible execution(*are messages delivered in every possible order?*) 
  is tested for safety & liveness properties.

* Sometimes the protocol authors make broad sketches, the gaps can be filled with confidence only through modelling.

* Testing the algorithm solution for safety & liveness properties helps in better algorithm design since the 
  solution is tested for correctness not for coverage as is usually done with unit & integration testing.
  
It needs to be mentioned that TLA+ comes with at least two challenges:   

* Modeling is only an approximate representation of the actual system, capturing the essential problems of the actual system
  into the model is challenging. 
  <br>For instance VSR paper is discussed using logs(*system state is represented as log & messages carry entire logs*) which is not practical. 
  In practical implementation, system state is composed of application state(commands in log applied to state machine) plus commands in the log, 
  log is pruned periodically. In reality, application state and log entries need to be exchanged while dealing with failure.  

* Model size needs to be limited as the state space grows exponentially. This creates additional tension during modeling.