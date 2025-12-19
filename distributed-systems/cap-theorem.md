# CAP Theorem & PACELC

<div align="center">

![Distributed](https://img.shields.io/badge/Distributed_Systems-CAP_Theorem-red?style=for-the-badge)
![Consistency](https://img.shields.io/badge/Models-Consistency_|_Availability_|_Partition-orange?style=for-the-badge)

*Understanding consistency models in distributed systems.*

</div>

## CAP Theorem

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAP THEOREM                                   │
│                                                                 │
│              Consistency (C)                                    │
│                   /\                                            │
│                  /  \                                           │
│                 /    \                                          │
│                /  CA  \     ← Not possible during partitions   │
│               /        \                                        │
│              /──────────\                                       │
│             /            \                                      │
│            /   CP    AP   \                                     │
│           /                \                                    │
│  Partition ─────────────────  Availability                     │
│  Tolerance (P)                (A)                               │
│                                                                 │
│  During a partition, choose:                                    │
│  • CP: Sacrifice availability for consistency                   │
│  • AP: Sacrifice consistency for availability                   │
└─────────────────────────────────────────────────────────────────┘
```

## Consistency Levels

```typescript
// Strong consistency (linearizability)
class LinearizableStore {
  private leader: Node;

  async write(key: string, value: unknown): Promise<void> {
    // All writes go through leader
    await this.leader.write(key, value);
    // Wait for majority acknowledgment
    await this.leader.waitForMajority();
  }

  async read(key: string): Promise<unknown> {
    // Read from leader only, or use read quorum
    return this.leader.read(key);
  }
}

// Eventual consistency
class EventuallyConsistentStore {
  private localReplica: Replica;

  async write(key: string, value: unknown): Promise<void> {
    // Write to local replica immediately
    await this.localReplica.write(key, value, Date.now());
    // Asynchronously replicate to others
    this.replicateAsync(key, value);
  }

  async read(key: string): Promise<unknown> {
    // Read from local replica (may be stale)
    return this.localReplica.read(key);
  }

  private async replicateAsync(key: string, value: unknown): Promise<void> {
    // Background replication with conflict resolution
    for (const peer of this.peers) {
      this.syncQueue.enqueue({ peer, key, value });
    }
  }
}

// Causal consistency
class CausallyConsistentStore {
  private vectorClock: Map<string, number> = new Map();

  async write(key: string, value: unknown, dependencies: VectorClock): Promise<void> {
    // Wait for dependencies to be visible
    await this.waitForDependencies(dependencies);

    // Increment our clock
    const nodeId = this.getNodeId();
    this.vectorClock.set(nodeId, (this.vectorClock.get(nodeId) ?? 0) + 1);

    // Store with vector clock
    await this.store.put(key, {
      value,
      vectorClock: new Map(this.vectorClock),
    });
  }

  async read(key: string): Promise<{ value: unknown; vectorClock: VectorClock }> {
    const result = await this.store.get(key);

    // Merge vector clocks
    this.mergeClocks(result.vectorClock);

    return result;
  }

  private async waitForDependencies(deps: VectorClock): Promise<void> {
    for (const [node, clock] of deps) {
      while ((this.vectorClock.get(node) ?? 0) < clock) {
        await this.pollForUpdates();
      }
    }
  }
}
```

## PACELC

```
┌─────────────────────────────────────────────────────────────────┐
│                         PACELC                                   │
│                                                                 │
│  IF there is a Partition (P):                                   │
│    Choose between Availability (A) and Consistency (C)          │
│                                                                 │
│  ELSE (normal operation):                                        │
│    Choose between Latency (L) and Consistency (C)               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Examples:                                                       │
│                                                                 │
│  PA/EL (DynamoDB, Cassandra):                                   │
│    Partition → Available, Normal → Low Latency                 │
│    (Accept stale reads for speed)                               │
│                                                                 │
│  PC/EC (VoltDB, HBase):                                         │
│    Partition → Consistent, Normal → Consistent                  │
│    (Always consistent, may block)                               │
│                                                                 │
│  PA/EC (MongoDB):                                                │
│    Partition → Available, Normal → Consistent                   │
│    (Strong consistency when healthy)                            │
│                                                                 │
│  PC/EL (PNUTS):                                                  │
│    Partition → Consistent, Normal → Low Latency                 │
│    (Fast reads with async replication)                          │
└─────────────────────────────────────────────────────────────────┘
```

## Quorum Patterns

```typescript
// Configurable quorum consistency
class QuorumStore {
  constructor(
    private replicas: Replica[],
    private writeQuorum: number,  // W
    private readQuorum: number,   // R
  ) {
    // W + R > N guarantees read-your-writes
    // W > N/2 guarantees no conflicting writes
    if (writeQuorum + readQuorum <= replicas.length) {
      console.warn('Quorum settings may allow stale reads');
    }
  }

  async write(key: string, value: unknown): Promise<void> {
    const timestamp = Date.now();
    const writePromises = this.replicas.map(r =>
      r.write(key, value, timestamp).catch(() => null)
    );

    const results = await Promise.all(writePromises);
    const successCount = results.filter(r => r !== null).length;

    if (successCount < this.writeQuorum) {
      throw new QuorumNotReachedError(`Only ${successCount}/${this.writeQuorum} writes succeeded`);
    }
  }

  async read(key: string): Promise<unknown> {
    const readPromises = this.replicas.map(r =>
      r.read(key).catch(() => null)
    );

    const results = await Promise.all(readPromises);
    const validResults = results.filter(r => r !== null);

    if (validResults.length < this.readQuorum) {
      throw new QuorumNotReachedError(`Only ${validResults.length}/${this.readQuorum} reads succeeded`);
    }

    // Return most recent value
    return this.selectMostRecent(validResults);
  }

  private selectMostRecent(results: ReadResult[]): unknown {
    return results.reduce((latest, current) =>
      current.timestamp > latest.timestamp ? current : latest
    ).value;
  }
}

// Sloppy quorum (hinted handoff)
class SloppyQuorumStore extends QuorumStore {
  async write(key: string, value: unknown): Promise<void> {
    const preferredNodes = this.getPreferenceList(key);
    let successCount = 0;
    const hints: HintedHandoff[] = [];

    for (const node of preferredNodes) {
      try {
        await node.write(key, value);
        successCount++;
      } catch {
        // Find another node and leave a hint
        const alternateNode = this.findAlternateNode(node);
        await alternateNode.writeHint(key, value, node.id);
        hints.push({ hintFor: node.id, storedAt: alternateNode.id });
        successCount++;
      }

      if (successCount >= this.writeQuorum) break;
    }

    if (successCount < this.writeQuorum) {
      throw new QuorumNotReachedError();
    }
  }

  // Background process to deliver hints
  async deliverHints(): Promise<void> {
    for (const node of this.replicas) {
      const hints = await node.getStoredHints();

      for (const hint of hints) {
        const targetNode = this.getNode(hint.targetNodeId);
        if (targetNode.isAvailable()) {
          await targetNode.write(hint.key, hint.value);
          await node.deleteHint(hint.id);
        }
      }
    }
  }
}
```

## Anti-Entropy & Merkle Trees

```typescript
// Merkle tree for efficient sync
class MerkleTree {
  private root: MerkleNode | null = null;

  buildFromData(data: Map<string, unknown>): void {
    const sortedKeys = Array.from(data.keys()).sort();
    const leaves = sortedKeys.map(key => ({
      key,
      hash: this.hash(JSON.stringify(data.get(key))),
    }));

    this.root = this.buildTree(leaves);
  }

  private buildTree(leaves: { key: string; hash: string }[]): MerkleNode {
    if (leaves.length === 1) {
      return { hash: leaves[0].hash, key: leaves[0].key };
    }

    const mid = Math.floor(leaves.length / 2);
    const left = this.buildTree(leaves.slice(0, mid));
    const right = this.buildTree(leaves.slice(mid));

    return {
      hash: this.hash(left.hash + right.hash),
      left,
      right,
    };
  }

  // Find differences between two trees
  findDifferences(other: MerkleTree): string[] {
    const differences: string[] = [];
    this.compareTrees(this.root, other.root, differences);
    return differences;
  }

  private compareTrees(
    a: MerkleNode | null,
    b: MerkleNode | null,
    differences: string[]
  ): void {
    if (!a && !b) return;

    if (!a || !b || a.hash !== b.hash) {
      // Trees differ
      if (a?.key) {
        differences.push(a.key);
      } else {
        this.compareTrees(a?.left ?? null, b?.left ?? null, differences);
        this.compareTrees(a?.right ?? null, b?.right ?? null, differences);
      }
    }
  }

  private hash(data: string): string {
    return crypto.createHash('sha256').update(data).digest('hex');
  }
}

// Anti-entropy protocol
class AntiEntropy {
  private merkleTree: MerkleTree;

  async sync(peer: Replica): Promise<void> {
    const peerTree = await peer.getMerkleTree();
    const differences = this.merkleTree.findDifferences(peerTree);

    if (differences.length === 0) {
      return; // Already in sync
    }

    // Only transfer differing keys
    const localData = await this.getKeys(differences);
    const peerData = await peer.getKeys(differences);

    // Merge using timestamps (LWW) or version vectors
    for (const key of differences) {
      const local = localData.get(key);
      const remote = peerData.get(key);

      const winner = this.resolveConflict(local, remote);

      if (winner === 'remote') {
        await this.store.put(key, remote!.value, remote!.timestamp);
      } else if (winner === 'local' && remote) {
        await peer.put(key, local!.value, local!.timestamp);
      }
    }

    // Rebuild Merkle tree
    this.rebuildMerkleTree();
  }

  private resolveConflict(
    local: VersionedValue | undefined,
    remote: VersionedValue | undefined
  ): 'local' | 'remote' | 'conflict' {
    if (!local) return 'remote';
    if (!remote) return 'local';

    // Last-Writer-Wins
    if (local.timestamp > remote.timestamp) return 'local';
    if (remote.timestamp > local.timestamp) return 'remote';

    // Tie-breaker using node ID
    return local.nodeId > remote.nodeId ? 'local' : 'remote';
  }
}

// Read repair
class ReadRepairStore extends QuorumStore {
  async read(key: string): Promise<unknown> {
    const results = await this.readFromAllReplicas(key);

    // Find most recent
    const mostRecent = this.selectMostRecent(results);

    // Repair stale replicas in background
    this.repairStaleReplicas(key, mostRecent, results);

    return mostRecent.value;
  }

  private async repairStaleReplicas(
    key: string,
    mostRecent: VersionedValue,
    results: { replica: Replica; value: VersionedValue }[]
  ): Promise<void> {
    const staleReplicas = results.filter(
      r => r.value.timestamp < mostRecent.timestamp
    );

    // Background repair
    Promise.all(
      staleReplicas.map(({ replica }) =>
        replica.write(key, mostRecent.value, mostRecent.timestamp)
      )
    ).catch(err => console.error('Read repair failed:', err));
  }
}
```

## Consistency Patterns in Practice

```typescript
// Multi-region deployment with consistency choices
class GlobalDataStore {
  constructor(
    private regions: Map<string, RegionalStore>,
    private config: ConsistencyConfig
  ) {}

  async write(key: string, value: unknown, options: WriteOptions): Promise<void> {
    const localRegion = this.getLocalRegion();

    switch (options.consistency) {
      case 'local':
        // Write to local region only (fastest)
        await this.regions.get(localRegion)!.write(key, value);
        break;

      case 'regional':
        // Synchronous write to local + async to others
        await this.regions.get(localRegion)!.write(key, value);
        this.replicateAsync(key, value, [localRegion]);
        break;

      case 'global':
        // Wait for majority of regions
        await this.writeToMajorityRegions(key, value);
        break;

      case 'strong':
        // Wait for all regions
        await Promise.all(
          Array.from(this.regions.values()).map(r => r.write(key, value))
        );
        break;
    }
  }

  async read(key: string, options: ReadOptions): Promise<unknown> {
    switch (options.consistency) {
      case 'eventual':
        // Read from local (may be stale)
        return this.regions.get(this.getLocalRegion())!.read(key);

      case 'session':
        // Read your writes within session
        return this.readWithSessionConsistency(key, options.sessionToken);

      case 'bounded':
        // Staleness within bounds
        return this.readWithBoundedStaleness(key, options.maxStalenessSeconds);

      case 'strong':
        // Read from leader or use read quorum
        return this.readFromLeader(key);
    }
  }
}
```

---

*Learned: December 20, 2025*
*Tags: Distributed Systems, CAP, Consistency, Quorum, Replication*
