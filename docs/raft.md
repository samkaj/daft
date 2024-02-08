# Raft?

Below are my notes from the famous [Raft paper](https://raft.github.io/raft.pdf), intended as a guide for me to ensure I understand what I am doing.

Consensus is an important concept in distributed systems, and it allows computers working together in synchrony as a result. Raft is a consensus algorithm, intended as a predecesor to Paxos with the goal of making it understandable, as Paxos is quite hard to grasp.

Raft has some key features:

- A strong leader: the elected leader acts as a source of truth, log entries flow from the leader to other servers.
- Leader election: choosing a leader is done with random intervals and a voting mechanism.
- Membership: when multiple clusters with different configurations overlap, joint consensus is applied

## Replicated state machines

State machines compute identical copies of the same state, which makes it runnable despite servers being down. Each server stores a replicated log which the state machine executes in order. Each log maintains a total order, making each state machine execute the commands in the same order, no matter where in the cluster the server might be.

Consensus is used for keeping the replicated log consistent. The module receives commands from clients and adds it to its log. It then communicates to the other servers which in turn adds it to theirs. After consensus is reached, the log is executed in order and sent as a response to the client. In this way, the cluster acts as one state machine when observed from outside.

## The algorithm

To gain consensus, a leader is first elected. The leader has the responsibility for managing the log. It accepts entries from clients, replicates them on other servers, and tells them when it is safe to apply it to their state machines. If the leader fails or disconnects, a new one is chosen.

### Basics

A cluster contains typically five servers. The majority needs to be functional, so two failed machines are allowed. There are three states a server can be in: `leader, follower, or candidate`. In normal operation, there is one leader and the rest are followers. The followers are passive, only responsing to calls from leaders and candidates. If a follower receives a request from a client, it simply forwards it to the leader.

The leader handles all client requests. The candidate state is used in leader elections. Time is divided into terms of random length. Terms are incremented, and every term is started with a leader election. On or more candidates try to get the leader role. When a candidate wins the elction, it serves as such for the rest of the term. In some situations, the votes are split, and those end without a leader, initiating a new term. This means that there is at most one leader in a term.

Terms are logical clocks, and help with detecting stale leaders, and every server maintains `currentTerm`, which can be used to detect if one is in an earlier term. If you detect that your term is less than another, you become a follower. Servers always communicate their current term to detect out of sync ones.

RPC is used for communications between servers, and two are only needed for consensus: `requestVote` and `appendEntries`. An additional third RPC is `snapshot`, which allows for transferring snapshots of the log. If an RPC fails, server retry if they timeout.

### Leader election

Hearbeats are used to initiate leader electinos. When servers start up, they always begin as followers, and remain in this state as long as they receive valid RPCs from leaders and candidates. Leaders send these heartbeats peridically to followers in order to maintain their position. Receivers that do not receive a heartbeat within a timeout, `electionTimeout`, deems it unviable, and starts a new election to choose a new leader.

#### Starting an election

A follower begins an election by incrementing its current term and transitions to the candidate states. It votes for itself and sends RPCs in parallel to each of the other server in the clusters. Candidates continue in this state until one of three things happen:

1. It wins the election and becomes leader
2. Another server becomes leader
3. No leader is elected
