# RaftKV - Distributed Key-Value Store with Raft Consensus

A production-ready, fault-tolerant distributed key-value store implementing the Raft consensus algorithm. This system provides strong consistency guarantees, automatic leader failover, and crash-safe persistence across a cluster of distributed servers.

> **Note**: This repository is private for academic integrity. This README serves as a portfolio showcase of the implementation.

## Contributors

This project was created by:

- **Dylan Pina** ([@DylanPina](https://github.com/DylanPina))
- **Will Sullivan** ([@wsulliv8](https://github.com/wsulliv8))

[![Go](https://img.shields.io/badge/Go-1.21+-00ADD8.svg)](https://golang.org/)
[![gRPC](https://img.shields.io/badge/gRPC-1.76+-4285F4.svg)](https://grpc.io/)
[![Protocol Buffers](https://img.shields.io/badge/Protocol%20Buffers-3.21+-4A90E2.svg)](https://protobuf.dev/)

## 🎯 Overview

RaftKV is a complete implementation of the Raft consensus protocol that enables a distributed key-value store to maintain consistency across multiple nodes, even in the presence of network partitions, server failures, and concurrent operations. The system guarantees that all committed operations are durable and that the cluster can recover from complete shutdowns.

### Key Features

- **Full Raft Consensus Implementation**: Complete leader election, log replication, and safety guarantees
- **Automatic Failover**: Sub-second leader election and seamless transition during failures
- **Crash-Safe Persistence**: Atomic writes with fsync guarantees for durable state storage
- **Network Partition Handling**: Correct behavior during split-brain scenarios and partition healing
- **Strong Consistency**: Linearizable operations with majority-based commit decisions
- **Production-Ready**: Comprehensive error handling, timeout management, and recovery mechanisms

## 🏗️ Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ gRPC
       ▼
┌─────────────────┐
│  Frontend       │  Port 8001
│  (Leader Proxy) │
└──────┬──────────┘
       │
       ├──────────────┬──────────────┬──────────────┐
       ▼              ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Server 0 │  │ Server 1 │  │ Server 2 │  │ Server N │
│ Port 9001│  │ Port 9002│  │ Port 9003│  │ Port 900N│
│          │  │          │  │          │  │          │
│ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │
│ │ Raft │ │  │ │ Raft │ │  │ │ Raft │ │  │ │ Raft │ │
│ │State │ │  │ │State │ │  │ │State │ │  │ │State │ │
│ └──────┘ │  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │
│ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │
│ │  KV  │ │  │ │  KV  │ │  │ │  KV  │ │  │ │  KV  │ │
│ │Store │ │  │ │Store │ │  │ │Store │ │  │ │Store │ │
│ └──────┘ │  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │
│ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │
│ │Persist││  │ │Persist││  │ │Persist││  │ │Persist││
│ └──────┘ │  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
```

The system consists of a frontend service that routes client requests to the appropriate Raft server, and a cluster of Raft servers that maintain replicated state through the consensus protocol. Each server maintains its own Raft state machine, key-value store, and persistent state.

## 📋 Core Components

### 1. Leader Election

- **Randomized Timeouts**: 150-300ms election timeouts prevent split votes
- **Term-Based Coordination**: Ensures at most one leader per term
- **Concurrent Vote Handling**: Thread-safe vote collection with proper synchronization
- **Election Safety**: Implements Raft's guarantee that at most one leader exists per term

### 2. Log Replication

- **Append-Only Log**: All operations are logged before execution
- **Majority Commit**: Entries committed only after replication to majority
- **Consistency Checking**: `prevLogIndex` and `prevLogTerm` ensure log matching
- **Conflict Resolution**: Automatic log truncation on inconsistencies
- **NextIndex Tracking**: Efficient log synchronization with follower-specific indices

### 3. State Machine

- **Deterministic Application**: Committed entries applied in strict order
- **Thread-Safe Operations**: Concurrent-safe state machine updates using mutexes
- **Read Consistency**: All servers serve reads from applied state
- **Last Applied Tracking**: Ensures entries are applied exactly once

### 4. Persistence

- **Atomic Writes**: Temp file → fsync → atomic rename pattern prevents corruption
- **Crash Safety**: State persisted before responding to clients
- **Full Recovery**: Cluster can recover from complete shutdown
- **State Files**: JSON-based storage with per-server state files
- **Persistent Fields**: Current term, voted-for, and complete log are all persisted

### 5. Fault Tolerance

- **Leader Failover**: Automatic election within seconds of leader failure
- **Network Partitions**: Correct behavior during split-brain scenarios
- **Server Recovery**: Seamless rejoin and log catch-up after restart
- **Minority Failures**: System continues operating with majority available
- **Split-Brain Prevention**: Only majority partitions can make progress

## 🔐 Safety Guarantees

RaftKV implements all five Raft safety properties:

1. **Election Safety**: At most one leader can be elected in a given term
2. **Leader Append-Only**: Leaders never overwrite or delete entries in their logs
3. **Log Matching**: If two logs contain an entry with the same index and term, they are identical up to that point
4. **Leader Completeness**: If a log entry is committed in a given term, it will be present in all future leaders
5. **State Machine Safety**: All servers apply the same sequence of log entries to their state machines

These guarantees ensure that the system maintains consistency even under concurrent operations, network partitions, and server failures.

## 🧪 Testing

The project includes comprehensive test suites covering:

- **Basic Operations**: Configuration validation, leader election, PUT/GET operations, log replication
- **Fault Tolerance**: Leader failure detection, server recovery, minority failures, split-brain prevention
- **Persistence**: State persistence, full cluster restart, network partitions, partition healing

All tests are language-agnostic and communicate with the servers via gRPC, ensuring the implementation works correctly across different failure scenarios.

## 🛠️ Technology Stack

- **Language**: Go 1.21+
- **RPC Framework**: gRPC with Protocol Buffers
- **Concurrency**: Goroutines with mutex-based synchronization
- **Persistence**: JSON-based state files with atomic writes
- **Testing**: Python-based language-agnostic test suite

## 📊 Performance Characteristics

- **Leader Election**: < 1 second in typical scenarios
- **Replication Latency**: ~100ms heartbeat interval
- **Write Throughput**: Limited by network latency and majority replication
- **Read Latency**: Sub-millisecond (local state machine access)
- **Recovery Time**: Seconds for full cluster restart

## 🔍 Implementation Highlights

### Concurrency Model

- Fine-grained locking with `sync.Mutex` for Raft state
- Separate locks for RPC clients to prevent deadlocks
- Channel-based notification for commit events
- Goroutine-based background tasks (heartbeats, log application)
- Careful lock ordering to prevent circular dependencies

### Persistence Strategy

```go
// Atomic write pattern ensures crash safety
tempFile := path + ".tmp"
Write(tempFile) → Sync() → Rename(tempFile, path)
```

This three-step process ensures that state files are never in a corrupted state, even if the process crashes mid-write. The atomic rename operation is guaranteed by the filesystem.

### Error Handling

- Graceful degradation during network failures
- Automatic retry with exponential backoff for RPC calls
- Context-based timeout management
- Comprehensive error logging for debugging
- Leadership verification before responding to clients

### Network Communication

- gRPC-based RPC for type-safe communication
- Protocol Buffers for efficient serialization
- Connection pooling and reuse
- Timeout handling for all network operations

## 📚 Project Structure

```
.
├── cmd/
│   ├── frontend/     # Frontend service entry point
│   └── server/       # Raft server entry point
├── internal/
│   ├── config/       # Configuration management
│   ├── frontend/     # Frontend service implementation
│   ├── raftpb/       # Protocol Buffer definitions
│   └── server/       # Raft server implementation
└── testsuite/        # Comprehensive test suites
```

The codebase follows Go best practices with clear separation of concerns, internal packages for implementation details, and a clean command structure.

## 🎓 Technical Achievements

This project demonstrates mastery of:

- **Distributed Systems Fundamentals**: Consensus algorithms, replication, fault tolerance, network partitions
- **Concurrency**: Thread-safe operations, race condition prevention, deadlock avoidance, lock ordering
- **System Design**: Service architecture, RPC communication, state management, persistence strategies
- **Production Engineering**: Error handling, logging, crash-safe persistence, comprehensive testing
- **Protocol Implementation**: Faithful implementation of the Raft consensus algorithm with all safety properties

## 📝 API Design

### Frontend Service

- `StartRaft(n)` - Initialize a fresh cluster with n servers
- `StartServer(id)` - Start a single server and add to cluster
- `Get(key)` - Retrieve value for key (any server)
- `Put(key, value)` - Store key-value pair (leader only)

### Server Service

- `ping()` - Health check endpoint
- `GetState()` - Return current Raft state (term, leader status, commit index)
- `Get(key)` - Read from local state machine
- `Put(key, value)` - Write operation (leader only)
- `AppendEntries(...)` - Raft log replication RPC
- `RequestVote(...)` - Raft leader election RPC

All operations are designed to be idempotent where possible, and the system handles leader changes gracefully by redirecting clients to the new leader.

## 🙏 Acknowledgments

- Raft paper by Diego Ongaro and John Ousterhout
- Protocol design based on the Raft consensus algorithm

---


