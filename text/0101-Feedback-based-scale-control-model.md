# Feedback-based scale control model

- RFC PR: https://github.com/tikv/rfcs/pull/101
- Tracking Issue: https://github.com/tikv/pd/issues/5467

## Summary

Each store has a separate sending window to limit the number of snapshot tasks, and the sending window size is 
dynamic by the snapshot's executing details. 

The fast snapshot task will get more sending window size, and the slow
snapshot task will get less sending window size.

It will reduce the waiting time and balance sending traffic to multiple stores at the same time. 

## Motivation

When the difference between the different region's size becomes bigger. It may bring following costs:
- There is a certain relationship between the speed and size of replica migration. By limiting the number of operators to control the replica speed. It cannot adapt to different region sizes, and cannot be associated with IO .
- The store limit only controls the storage of the source and target instance. In fact, the speed of the operator is mainly affected by the sender and the receiver, so it may cause a serious imbalance in the sending tasks of the individual nodes.
- The TIKV executes snapshot tasks by FIFO. When the consumption speed of the sender is lower than that of the operator producer, it will take unnecessary wait duration, thereby affecting the timeliness of the operator. So the task will be held longer and other scheduler cannot scheduler this region. 

Hence, I propose using a new flow mechanism to control the concurrent flow of the sender. 
It will be helpful to solve the above problems.

## Detailed Design

The key point of the problem is to balance the speed between the operator producers and the operator consumers. 

And the operator consumer can be divided into the snapshot sender (region leader ) and receiver (new peer). 

PD can determine the operator's type and quantity according to the current task execution situation. 

### Flow Mechanism

In order to prevent PD from over-delivering snapshot tasks to the TIKV instance. 

Each store will have sliding windows to send snapshots, and the total size of the sliding window can be adjusted 
by the snapshot execution duration and the total execution duration. 

```mermaid
graph LR
    subgraph PD
            subgraph Store
                RS[regions]
                SW[sliding window]
            end
      SC[Scheduler]-->|1. pick region|RS
      RS -->|2. check store|SW
            RS -->|3. region available|SC 
      SC -->|4. put operator and take token| OC[operator controller]    
    end 

    
    
    subgraph TIKV-leader
      OC -.->|5. region heartbeat| R[Raft]
      R -.->|6. generate task| P[pool]
      P -->|FIFO|P
      P -.->|7. send task| SE[sender]
            P -->|7.2 generate duration +|Z
            SE-->|8.2 send duration +|Z[collect]
      
      Z -.->|9.2 store heartbeat|SW
    end 
    
    subgraph TIKV-learner
        SE -->|8. send snapshot|RE[receiver]
     RE-->|9. apply snapshot|R1[raft]
    end
```

The mechanism of the scheduling limiter is shown in the figure. 

All scheduler will check the region that's leader has some available send size, if not, it will try another region as target.

### How to speed up/down scheduling  

The current users can speed up/down scheduling though store limit API.

In order to reduce the user's cost, the new current limit will inherit this API and switch mode through the configuration. 

The new API may look like this:

```
config set limit v2 // Switch flow limit v2
store limit all 500 // All stores can at most move 500MB data per second 
store limit 1 200   // Store 1 can at most move 200MB data per second
```


### How to adjust windows size 

The minimum(initial) value of the sliding window may be 100MB.

The store can be considered to be scheduled if it's available sending size is bigger than 100MB, so the biggest region can also be scheduled. 

When the operator execution duration is less than the given times(default is 2) the snapshot task execution time, the sender windows size will increase, otherwise it will decrease. 

A PI controller is used to prevent repeated adjustments, it should be noted that when all schedules don't need such big size, it will stop increasing. 

### How to ensure fairness 

The new scheduler policy does not guarantee that the snapshot sending size of all stores is exactly the same, but will switch to the next candidate region when the store has no available sending size. 

During the prototype testing process on the scale-out and scale-in scene , the sending size on all TIKV reaches the bottleneck. 

### Scheduler 
#### Balance Region Scheduler 

The scheduler picks the store as source first, then picks the region on the source store, finally picks the target store. On the original basis, it is necessary to check the target region leader whether there is enough sending space to send. 

#### High Priority Scheduler 

Some scheduler may create the highest priority operator such as balance hot region, region check. For the highest operators (such as hot region operator), they can use this external space if the normal space is not enough. The default external space is about 20% of the send window. This can prevent this window size from being completely occupied by low priority task.

```
|------- Used -------|
                     |--------Available -----------|
|------ Normal ------------|---- Privilege —-------|
```
As shown in the figure, the normal space is used to schedule the normal operator, and the privilege space is used to schedule the high priority operator.

In current, the privilege space can be devided into four parts: Low, Normal, High, Urgent.


## Compatibility

### Store Limit

The store limiter mainly has the following functions that cannot be replaced:
- The second highest (lowest) score store can still be scheduled. An instance cannot continuously be the source or target to be scheduler because the token is limited in the short term.
- Remove too many peers from the same store can bring major compaction, which affect the cluster performance.

### Multi RocksDB

In this design, the generation of the snapshot will use check-point hard link to replace the scanning of RocksDB. 

It can reduce the CPU loads and the duration of snapshot generator, so the executing duration of snapshot most spent on the network.

The snapshot synchronization process is basically the same, some tasks need to wait to be executed when the store has many snapshot tasks. 

The waiting time is still positively related to the backlog of the snapshot task, so feedback mechanism still satisfies this situation. 

## Alternatives

### [Follower Replica](https://github.com/tikv/rfcs/pull/98)

A new mechanism in the raft algorithm will be introduced which allows the follower to replica the log or send snapshot to another peer. 

Since the removed peer can also be the snapshot sender, balance region scheduler no need to consult the region leader whether there is some available sending size. 

But for the other scheduler, there may be a big log gap between the follower and leader, so the follower cannot be the snapshot sender, this may make the operator slower and complicated.

## Drawbacks

Due to the need to prevent too many snapshot tasks on the instance, a new limit needs to be added, which may cause the execution priority of the scheduler to change, resulting in a small number of scheduling. 

## Question

## Unresolved questions