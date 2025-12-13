# Go-Raft: Distributed Key-Value Store

Academic Integrity Notice

This repository serves as a portfolio piece documenting the architecture, design specifications, and performance characteristics of a Raft Consensus Protocol implementation. The source code is hosted in a private repository to comply with university academic integrity policies. Access to the full source code (Go, gRPC definitions, and test suite) can be granted to interviewers and hiring managers upon request.

---

## Project Overview

Go-Raft is a CP-system (Consistent, Partition-Tolerant) distributed key-value store engineered in Golang. It implements the Raft Consensus Algorithm to manage replicated logs across a cluster of nodes, ensuring linearizable consistency and data durability.

The system is designed to maintain availability during leader failures and network partitions, automatically resolving split-brain scenarios through majority quorum voting. It serves as a foundational infrastructure component similar to etcd or Consul, supporting atomic `GET` and `PUT` operations via a gRPC API.

---

## System Architecture

The cluster architecture consists of five distinct server nodes and a frontend gateway, communicating exclusively via gRPC with Protocol Buffers.

### 1. Frontend Service (Gateway)

The frontend acts as a smart reverse proxy and client entry point listening on port 8001.

- **Service Discovery:** Parses the cluster configuration to identify active node addresses.
    
- **Leader Routing:** Queries the cluster state to identify the current leader and forwards client `Put` and `Get` requests to the appropriate node.
    
- **Retry Logic:** Implements backoff and retry mechanisms to handle transient leader elections or network timeouts during forwarding.
    

### 2. Consensus Engine (Raft Nodes)

Each node (listening on ports 9001-9005) operates an autonomous state machine based on the Raft specification.

- **State Machine:** Nodes transition between **Follower**, **Candidate**, and **Leader** states based on election timeouts and RPC inputs.
    
- **Concurrency Model:** Utilizes Go channels and goroutines to handle blocking RPCs (AppendEntries, RequestVote) asynchronously from the main event loop, preventing heartbeat starvation.
    

### 3. Storage Layer (WAL)

Data durability is achieved through a Write-Ahead Log (WAL) persisted to the local filesystem.

- **Atomic Persistence:** Critical state (`currentTerm`, `votedFor`, and `log[]`) is flushed to disk using atomic file operations (write-to-temp followed by `rename`) before any RPC response is sent, ensuring crash consistency.
    

---

## Technical Implementation Details

### Leader Election and Livelock Prevention

To ensure high availability, the system implements a robust leader election mechanism.

- **Randomized Timeouts:** To prevent "split votes"—where nodes repeatedly time out simultaneously and fail to elect a leader—the election timeout is randomized between 150ms and 300ms.
    
- **Term Logic:** Nodes strictly adhere to term monotonicity. If a Candidate or Leader discovers a peer with a higher term, it immediately reverts to the Follower state to prevent stale leadership.
    

### Log Replication and Consistency

The system guarantees that once a log entry is committed, it will be present on all future leaders.

- **Quorum Commit:** A `PUT` operation is only acknowledged to the client once the log entry has been replicated to a majority (3 out of 5) of nodes.
    
- **Log Matching:** The `AppendEntries` RPC enforces consistency by including `prevLogIndex` and `prevLogTerm`. Followers reject entries that do not align with their local history, triggering a reconciliation process where the leader decrements its `nextIndex` for that follower until a matching point is found.
    
- **No-Op Commits:** Upon election, a new leader commits a "no-op" entry to assert authority and indirectly commit entries from previous terms, ensuring the `commitIndex` advances correctly.
    

### Fault Tolerance and Partition Handling

The system is tested to survive specific failure modes described in the design specification.

- **Leader Failover:** The cluster detects leader crashes via missing heartbeats and elects a new leader within the election timeout window.
    
- **Split-Brain Resolution:** In the event of a network partition (e.g., a 3-node vs. 2-node split), the minority partition automatically ceases to accept writes because it cannot achieve a quorum. The majority partition continues to operate, preserving consistency.
    
- **Partition Healing:** When network connectivity is restored, nodes in the minority partition recognize the higher term from the majority leader, step down, and backfill their logs to match the authoritative state.
    

### Disaster Recovery

- **Full Cluster Restart:** The system supports a complete shutdown and restart. Nodes reload their persistent state from disk upon boot, reconstruct their volatile state machines (Hash Maps), and resume operations without data loss.
    
- **Crash Recovery:** Individual nodes can be restarted via the `StartServer` RPC. They automatically rejoin the cluster and catch up on missed log entries via the Leader's replication mechanism.
    

---

## Performance & Testing

The implementation was validated against a strict test suite covering correctness and resilience scenarios.

|**Test Category**|**Description**|**Status**|
|---|---|---|
|**Basic KV Operations**|Verified linearizable Get/Put operations on a healthy cluster.|Passed|
|**Leader Failure**|Verified continuous availability during forced leader process termination.|Passed|
|**Minority Partition**|Confirmed writes are rejected when <3 nodes are reachable.|Passed|
|**Majority Partition**|Confirmed writes succeed in the partition containing 3 nodes.|Passed|
|**Persistence**|Verified data integrity across multiple full-cluster restarts.|Passed|
|**Concurrency**|Stress tested with concurrent client requests to identify race conditions.|Passed|

---

## Deployment Configuration

The system is configured via a standard `config.ini` file, allowing for flexible deployment topologies.

**Configuration Parameters:**

- `base_address`: Localhost or IP address for binding.
    
- `base_port`: Starting port for Raft peer communication (default: 9001).
    
- `persistent_state_path`: Directory path for WAL storage (e.g., `raft_state/`).
    

Run Instructions (Docker):

The project includes a Dockerfile and docker-compose.yml to orchestrate the 5-node cluster and frontend service.

Bash

```
# Build the Go binaries
make build

# Start the cluster
docker-compose up -d

# Check cluster status via logs
docker-compose logs -f
```

---

## Contact

If you would like to discuss the architectural decisions, view the private source code, or see a demonstration of the failure recovery mechanisms, please contact me.

- **Email:** [Your Email]
    
- **LinkedIn:** [Your Profile]
    
- **Portfolio:** [Your Website]