# Consensus Protocols

Distributed agreement algorithms for fault-tolerant systems.

## Raft Consensus

```
┌─────────────────────────────────────────────────────────────────┐
│                    RAFT STATE MACHINE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐      timeout      ┌───────────┐                │
│   │ Follower │ ─────────────────→│ Candidate │                 │
│   └──────────┘                   └───────────┘                 │
│        ↑                              │                         │
│        │ discovers higher term        │ receives majority      │
│        │ or leader heartbeat          │ votes                  │
│        │                              ↓                         │
│        │                        ┌──────────┐                   │
│        └────────────────────────│  Leader  │                   │
│                                 └──────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```typescript
// Simplified Raft implementation
enum NodeState {
  Follower = 'follower',
  Candidate = 'candidate',
  Leader = 'leader',
}

interface LogEntry {
  term: number;
  index: number;
  command: unknown;
}

class RaftNode {
  private state: NodeState = NodeState.Follower;
  private currentTerm = 0;
  private votedFor: string | null = null;
  private log: LogEntry[] = [];
  private commitIndex = 0;
  private lastApplied = 0;

  // Leader state
  private nextIndex: Map<string, number> = new Map();
  private matchIndex: Map<string, number> = new Map();

  private electionTimeout: NodeJS.Timer | null = null;
  private heartbeatInterval: NodeJS.Timer | null = null;

  constructor(
    private readonly nodeId: string,
    private readonly peers: string[],
    private readonly rpc: RaftRPC,
    private readonly stateMachine: StateMachine
  ) {
    this.resetElectionTimeout();
  }

  // Leader election
  private startElection(): void {
    this.state = NodeState.Candidate;
    this.currentTerm++;
    this.votedFor = this.nodeId;

    let votesReceived = 1; // Vote for self
    const votesNeeded = Math.floor(this.peers.length / 2) + 1;

    const lastLogIndex = this.log.length - 1;
    const lastLogTerm = this.log[lastLogIndex]?.term ?? 0;

    for (const peer of this.peers) {
      this.rpc.requestVote(peer, {
        term: this.currentTerm,
        candidateId: this.nodeId,
        lastLogIndex,
        lastLogTerm,
      }).then(response => {
        if (response.term > this.currentTerm) {
          this.stepDown(response.term);
          return;
        }

        if (response.voteGranted) {
          votesReceived++;
          if (votesReceived >= votesNeeded && this.state === NodeState.Candidate) {
            this.becomeLeader();
          }
        }
      });
    }
  }

  private becomeLeader(): void {
    this.state = NodeState.Leader;
    this.stopElectionTimeout();

    // Initialize leader state
    for (const peer of this.peers) {
      this.nextIndex.set(peer, this.log.length);
      this.matchIndex.set(peer, 0);
    }

    // Start heartbeats
    this.heartbeatInterval = setInterval(() => this.sendHeartbeats(), 50);
    this.sendHeartbeats();
  }

  private sendHeartbeats(): void {
    for (const peer of this.peers) {
      this.sendAppendEntries(peer);
    }
  }

  private async sendAppendEntries(peer: string): Promise<void> {
    const nextIdx = this.nextIndex.get(peer) ?? this.log.length;
    const prevLogIndex = nextIdx - 1;
    const prevLogTerm = this.log[prevLogIndex]?.term ?? 0;
    const entries = this.log.slice(nextIdx);

    const response = await this.rpc.appendEntries(peer, {
      term: this.currentTerm,
      leaderId: this.nodeId,
      prevLogIndex,
      prevLogTerm,
      entries,
      leaderCommit: this.commitIndex,
    });

    if (response.term > this.currentTerm) {
      this.stepDown(response.term);
      return;
    }

    if (response.success) {
      this.nextIndex.set(peer, nextIdx + entries.length);
      this.matchIndex.set(peer, nextIdx + entries.length - 1);
      this.updateCommitIndex();
    } else {
      // Decrement nextIndex and retry
      this.nextIndex.set(peer, Math.max(0, nextIdx - 1));
    }
  }

  // Log replication
  async appendCommand(command: unknown): Promise<boolean> {
    if (this.state !== NodeState.Leader) {
      return false;
    }

    const entry: LogEntry = {
      term: this.currentTerm,
      index: this.log.length,
      command,
    };

    this.log.push(entry);

    // Replicate to followers
    await this.replicateToMajority(entry);

    return true;
  }

  private async replicateToMajority(entry: LogEntry): Promise<void> {
    const replicationPromises = this.peers.map(peer =>
      this.sendAppendEntries(peer)
    );

    await Promise.all(replicationPromises);
    this.updateCommitIndex();
  }

  private updateCommitIndex(): void {
    const matchIndices = [this.log.length - 1, ...Array.from(this.matchIndex.values())];
    matchIndices.sort((a, b) => b - a);

    const majorityIndex = matchIndices[Math.floor(matchIndices.length / 2)];

    if (majorityIndex > this.commitIndex && this.log[majorityIndex]?.term === this.currentTerm) {
      this.commitIndex = majorityIndex;
      this.applyCommitted();
    }
  }

  private applyCommitted(): void {
    while (this.lastApplied < this.commitIndex) {
      this.lastApplied++;
      const entry = this.log[this.lastApplied];
      this.stateMachine.apply(entry.command);
    }
  }

  // Handle incoming RPCs
  handleRequestVote(request: RequestVoteRequest): RequestVoteResponse {
    if (request.term < this.currentTerm) {
      return { term: this.currentTerm, voteGranted: false };
    }

    if (request.term > this.currentTerm) {
      this.stepDown(request.term);
    }

    const logOk = this.isLogUpToDate(request.lastLogIndex, request.lastLogTerm);
    const canVote = this.votedFor === null || this.votedFor === request.candidateId;

    if (canVote && logOk) {
      this.votedFor = request.candidateId;
      this.resetElectionTimeout();
      return { term: this.currentTerm, voteGranted: true };
    }

    return { term: this.currentTerm, voteGranted: false };
  }

  handleAppendEntries(request: AppendEntriesRequest): AppendEntriesResponse {
    if (request.term < this.currentTerm) {
      return { term: this.currentTerm, success: false };
    }

    this.resetElectionTimeout();

    if (request.term > this.currentTerm || this.state !== NodeState.Follower) {
      this.stepDown(request.term);
    }

    // Check log consistency
    if (request.prevLogIndex >= 0) {
      const prevEntry = this.log[request.prevLogIndex];
      if (!prevEntry || prevEntry.term !== request.prevLogTerm) {
        return { term: this.currentTerm, success: false };
      }
    }

    // Append entries
    for (let i = 0; i < request.entries.length; i++) {
      const logIndex = request.prevLogIndex + 1 + i;
      const existing = this.log[logIndex];

      if (existing && existing.term !== request.entries[i].term) {
        // Conflict: delete existing and all following
        this.log.splice(logIndex);
      }

      if (!this.log[logIndex]) {
        this.log.push(request.entries[i]);
      }
    }

    // Update commit index
    if (request.leaderCommit > this.commitIndex) {
      this.commitIndex = Math.min(request.leaderCommit, this.log.length - 1);
      this.applyCommitted();
    }

    return { term: this.currentTerm, success: true };
  }

  private stepDown(newTerm: number): void {
    this.currentTerm = newTerm;
    this.state = NodeState.Follower;
    this.votedFor = null;
    this.stopHeartbeat();
    this.resetElectionTimeout();
  }

  private resetElectionTimeout(): void {
    this.stopElectionTimeout();
    const timeout = 150 + Math.random() * 150; // 150-300ms
    this.electionTimeout = setTimeout(() => this.startElection(), timeout);
  }

  private stopElectionTimeout(): void {
    if (this.electionTimeout) {
      clearTimeout(this.electionTimeout);
      this.electionTimeout = null;
    }
  }

  private stopHeartbeat(): void {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
      this.heartbeatInterval = null;
    }
  }

  private isLogUpToDate(lastLogIndex: number, lastLogTerm: number): boolean {
    const myLastIndex = this.log.length - 1;
    const myLastTerm = this.log[myLastIndex]?.term ?? 0;

    if (lastLogTerm !== myLastTerm) {
      return lastLogTerm > myLastTerm;
    }
    return lastLogIndex >= myLastIndex;
  }
}
```

## Paxos

```typescript
// Multi-Paxos for state machine replication
interface Proposal {
  proposalNumber: ProposalNumber;
  value: unknown;
}

interface ProposalNumber {
  round: number;
  nodeId: string;
}

class PaxosNode {
  private highestProposalSeen: ProposalNumber | null = null;
  private acceptedProposal: Proposal | null = null;
  private decidedValue: unknown = null;

  // Proposer
  async propose(value: unknown): Promise<unknown> {
    const proposalNumber = this.generateProposalNumber();

    // Phase 1: Prepare
    const promises = await this.sendPrepare(proposalNumber);
    const majority = Math.floor(this.peers.length / 2) + 1;

    if (promises.length < majority) {
      throw new Error('Failed to get majority in prepare phase');
    }

    // Use highest accepted value from promises, or our value
    const valueToPropose = this.selectValue(promises, value);

    // Phase 2: Accept
    const accepts = await this.sendAccept(proposalNumber, valueToPropose);

    if (accepts.length < majority) {
      throw new Error('Failed to get majority in accept phase');
    }

    // Decided
    this.decidedValue = valueToPropose;
    await this.broadcastDecided(valueToPropose);

    return valueToPropose;
  }

  private async sendPrepare(proposalNumber: ProposalNumber): Promise<PrepareResponse[]> {
    const responses = await Promise.all(
      this.peers.map(peer =>
        this.rpc.prepare(peer, { proposalNumber }).catch(() => null)
      )
    );

    return responses.filter((r): r is PrepareResponse => r !== null && r.promised);
  }

  private async sendAccept(
    proposalNumber: ProposalNumber,
    value: unknown
  ): Promise<AcceptResponse[]> {
    const responses = await Promise.all(
      this.peers.map(peer =>
        this.rpc.accept(peer, { proposalNumber, value }).catch(() => null)
      )
    );

    return responses.filter((r): r is AcceptResponse => r !== null && r.accepted);
  }

  // Acceptor
  handlePrepare(request: PrepareRequest): PrepareResponse {
    if (this.isHigher(request.proposalNumber, this.highestProposalSeen)) {
      this.highestProposalSeen = request.proposalNumber;

      return {
        promised: true,
        acceptedProposal: this.acceptedProposal,
      };
    }

    return { promised: false };
  }

  handleAccept(request: AcceptRequest): AcceptResponse {
    if (this.isHigherOrEqual(request.proposalNumber, this.highestProposalSeen)) {
      this.highestProposalSeen = request.proposalNumber;
      this.acceptedProposal = {
        proposalNumber: request.proposalNumber,
        value: request.value,
      };

      return { accepted: true };
    }

    return { accepted: false };
  }

  private selectValue(promises: PrepareResponse[], defaultValue: unknown): unknown {
    const accepted = promises
      .filter(p => p.acceptedProposal !== null)
      .map(p => p.acceptedProposal!);

    if (accepted.length === 0) {
      return defaultValue;
    }

    // Return value with highest proposal number
    return accepted.reduce((highest, current) =>
      this.isHigher(current.proposalNumber, highest.proposalNumber) ? current : highest
    ).value;
  }

  private generateProposalNumber(): ProposalNumber {
    const round = this.highestProposalSeen
      ? this.highestProposalSeen.round + 1
      : 1;
    return { round, nodeId: this.nodeId };
  }

  private isHigher(a: ProposalNumber | null, b: ProposalNumber | null): boolean {
    if (!a) return false;
    if (!b) return true;
    if (a.round !== b.round) return a.round > b.round;
    return a.nodeId > b.nodeId;
  }

  private isHigherOrEqual(a: ProposalNumber | null, b: ProposalNumber | null): boolean {
    if (!a) return false;
    if (!b) return true;
    if (a.round !== b.round) return a.round >= b.round;
    return a.nodeId >= b.nodeId;
  }
}
```

## Byzantine Fault Tolerance (PBFT)

```typescript
// Practical Byzantine Fault Tolerance
class PBFTNode {
  private view = 0;
  private sequence = 0;
  private prepareMessages: Map<string, Set<string>> = new Map();
  private commitMessages: Map<string, Set<string>> = new Map();

  // f = max faulty nodes, n = total nodes
  // n >= 3f + 1
  private readonly f: number;
  private readonly quorumSize: number;

  constructor(
    private readonly nodeId: string,
    private readonly nodes: string[],
    private readonly rpc: PBFTRPC
  ) {
    this.f = Math.floor((nodes.length - 1) / 3);
    this.quorumSize = 2 * this.f + 1;
  }

  // Primary receives client request
  async handleClientRequest(request: ClientRequest): Promise<void> {
    if (!this.isPrimary()) {
      // Forward to primary
      await this.rpc.forward(this.getPrimary(), request);
      return;
    }

    // Assign sequence number
    const message: PrePrepareMessage = {
      view: this.view,
      sequence: this.sequence++,
      digest: this.digest(request),
      request,
    };

    // Broadcast PRE-PREPARE
    await this.broadcastPrePrepare(message);
  }

  // Backup receives PRE-PREPARE
  async handlePrePrepare(message: PrePrepareMessage): Promise<void> {
    // Validate
    if (message.view !== this.view) return;
    if (!this.isFromPrimary(message)) return;
    if (this.hasConflictingPrePrepare(message)) return;

    // Store and broadcast PREPARE
    this.storePrePrepare(message);

    const prepare: PrepareMessage = {
      view: message.view,
      sequence: message.sequence,
      digest: message.digest,
      nodeId: this.nodeId,
    };

    await this.broadcast('prepare', prepare);
  }

  // Node receives PREPARE
  handlePrepare(message: PrepareMessage): void {
    const key = this.messageKey(message);

    if (!this.prepareMessages.has(key)) {
      this.prepareMessages.set(key, new Set());
    }

    this.prepareMessages.get(key)!.add(message.nodeId);

    // Check if prepared (2f prepares + pre-prepare)
    if (this.isPrepared(message)) {
      this.broadcastCommit(message);
    }
  }

  // Node receives COMMIT
  handleCommit(message: CommitMessage): void {
    const key = this.messageKey(message);

    if (!this.commitMessages.has(key)) {
      this.commitMessages.set(key, new Set());
    }

    this.commitMessages.get(key)!.add(message.nodeId);

    // Check if committed-local (2f + 1 commits)
    if (this.isCommittedLocal(message)) {
      this.executeRequest(message);
    }
  }

  private isPrepared(message: PrepareMessage): boolean {
    const key = this.messageKey(message);
    const prepareCount = this.prepareMessages.get(key)?.size ?? 0;
    return prepareCount >= this.quorumSize && this.hasMatchingPrePrepare(message);
  }

  private isCommittedLocal(message: CommitMessage): boolean {
    const key = this.messageKey(message);
    const commitCount = this.commitMessages.get(key)?.size ?? 0;
    return commitCount >= this.quorumSize && this.isPrepared(message);
  }

  // View change protocol
  async initiateViewChange(): Promise<void> {
    const viewChange: ViewChangeMessage = {
      newView: this.view + 1,
      nodeId: this.nodeId,
      // Include prepared messages as proof
      preparedProofs: this.getPrepareds(),
    };

    await this.broadcast('view-change', viewChange);
  }

  private isPrimary(): boolean {
    return this.getPrimary() === this.nodeId;
  }

  private getPrimary(): string {
    return this.nodes[this.view % this.nodes.length];
  }

  private messageKey(msg: { view: number; sequence: number; digest: string }): string {
    return `${msg.view}:${msg.sequence}:${msg.digest}`;
  }
}
```

---

*Learned: 2024*
*Tags: Distributed Systems, Raft, Paxos, PBFT, Consensus, Replication*
