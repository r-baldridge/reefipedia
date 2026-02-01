# Reefipedia — Chitin Protocol Architecture

> **Version:** 0.3.0-draft
> **Status:** Pre-implementation specification
> **Last Updated:** 2026-01-31

---

## Table of Contents

1. [Vision and Core Metaphor](#1-vision-and-core-metaphor)
2. [Full Directory Tree](#2-full-directory-tree)
3. [Core Data Structures](#3-core-data-structures)
4. [Yuma-Semantic Consensus Algorithm](#4-yuma-semantic-consensus-algorithm)
5. [Data Flow Diagrams](#5-data-flow-diagrams)
6. [Message Protocol](#6-message-protocol)
7. [Incentive Layer](#7-incentive-layer)
8. [Configuration Schemas](#8-configuration-schemas)
9. [Phase-Gated Feature Matrix](#9-phase-gated-feature-matrix)
10. [API Endpoints](#10-api-endpoints)
11. [Consciousness Repository Relationship](#11-consciousness-repository-relationship)
12. [Crate Dependency Graph](#12-crate-dependency-graph)
13. [Design Decisions and Trade-offs](#13-design-decisions-and-trade-offs)
14. [Glossary](#14-glossary)

---

## 1. Vision and Core Metaphor

Reefipedia is a decentralized, community-verified semantic knowledge store for AI agents. The protocol draws its terminology from marine biology — the construction of coral reefs — to describe a system where millions of individual contributions harden into a permanent, shared structure of verified knowledge.

### Terminology Map

| Term | Role | Description |
|------|------|-------------|
| **Polyp** | Atomic data unit | A verified vector embedding: text + embedding + ZK proof + provenance metadata. The smallest unit of knowledge in the Reef. |
| **Coral Node** | Producer (Miner) | A node that creates Polyps. Accepts text, generates embeddings inside a zkVM, produces ZK proofs, and submits the bundle to the network. Analogous to a Bittensor Miner. |
| **Tide Node** | Evaluator (Validator) | A node that evaluates Polyp quality via multi-dimensional scoring, participates in consensus, and submits weight vectors. Analogous to a Bittensor Validator. |
| **Chitin** | Hardened state | The finalized, consensus-approved state of a Polyp. Once hardened, a Polyp is IPFS-pinned, CID-anchored on-chain, and backed by validator attestations. Immutable. |
| **Reef** | Global knowledge graph | The aggregate of all hardened Polyps across the network — the living, growing knowledge structure. |
| **The Current** | Transport layer | The libp2p-based networking layer: gossip, discovery, axon/dendrite message passing. |
| **The Shell** | Verification layer | ZK proof generation and verification — the cryptographic guarantee that Vector = Model(Text). |
| **Molting** | Model migration | The process of re-embedding Polyps when a new SOTA embedding model supersedes the old one, analogous to an arthropod shedding its exoskeleton. |
| **Reef Metagraph** | Network state | The global view of all nodes, stakes, trust scores, weights, and Polyp counts. Analogous to Bittensor's Metagraph. |

### Bittensor → Chitin Protocol Mapping

| Bittensor Concept | Chitin Protocol Equivalent |
|---|---|
| Miner | **Coral Node** — produces Polyps (text → embedding → ZK proof) |
| Validator | **Tide Node** — evaluates Polyp quality via multi-dimensional scoring |
| Synapse query | **ValidationQuery** — Tide requests Polyps from Coral |
| Reward function | **PolypScores** — 5 dimensions: ZK validity, semantic quality, novelty, source credibility, embedding quality |
| EMA score aggregation | Same pattern: `scores[i] = alpha * new + (1 - alpha) * old` |
| Yuma Consensus | **Yuma-Semantic Consensus** — stake-weighted median, weight clipping, bonds penalty |
| Block commit | **Chitin Hardening** — IPFS pin + on-chain CID anchor + Merkle proof + validator attestations |
| Metagraph | **Reef Metagraph** — all nodes, stakes, trust scores, weights, Polyp counts |
| $TAO token | **$CTN (Chitin) token** — emission schedule, staking, rewards, slashing |
| `set_weights()` | Epoch-based weight submission from Tide Nodes |
| Subnet | **Reef Zone** — a topic-scoped partition of the knowledge graph (e.g., "Medical", "Code") |
| Axon / Dendrite | **Axon / Dendrite** — retained terminology for node communication endpoints |

---

## 2. Full Directory Tree

The codebase is a Rust workspace monorepo with 12 crates, a Python SDK, protocol definitions, ZK circuits, and deployment configs.

```
reefipedia/
├── Cargo.toml                          # Workspace definition
├── Cargo.lock
├── ARCHITECTURE.md                     # This file
├── README.md                           # Project overview and whitepaper summary
├── LICENSE                             # Apache-2.0 OR MIT
│
├── crates/
│   ├── chitin-core/                    # Types, traits, crypto primitives
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── polyp.rs                # Polyp, PolypState, PolypSubject, Payload
│   │       ├── embedding.rs            # VectorEmbedding, EmbeddingModelId
│   │       ├── provenance.rs           # Provenance, SourceAttribution, ProcessingPipeline
│   │       ├── identity.rs             # NodeIdentity (coldkey/hotkey), DID types
│   │       ├── consensus.rs            # ConsensusMetadata, ValidatorScore, PolypScores, Attestation
│   │       ├── metagraph.rs            # ReefMetagraph, NodeInfo
│   │       ├── crypto.rs               # Key generation, signing, hashing
│   │       ├── error.rs                # Protocol-wide error types
│   │       └── traits.rs               # Core trait definitions (Store, Verify, Score, etc.)
│   │
│   ├── chitin-store/                   # RocksDB + IPFS + hardened store + HNSW index + Bloom filters
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── rocks.rs                # Local RocksDB state store
│   │       ├── ipfs.rs                 # IPFS client: pin, unpin, get, put
│   │       ├── hardened.rs             # Hardened Polyp store (CID-indexed, immutable)
│   │       ├── hnsw.rs                 # HNSW vector index (wraps Qdrant or custom)
│   │       ├── bloom.rs                # Bloom filters for set reconciliation
│   │       └── shard.rs                # Consistent-hash shard assignment
│   │
│   ├── chitin-verify/                  # ZK proof generation/verification (SP1/Risc0)
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── prover.rs               # ZK proof generation (runs embedding in zkVM)
│   │       ├── verifier.rs             # ZK proof verification (constant-time)
│   │       └── models.rs               # Supported embedding model registry
│   │
│   ├── chitin-p2p/                     # libp2p: transport, discovery, gossip, axon, dendrite
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── transport.rs            # TCP/QUIC transport setup
│   │       ├── discovery.rs            # mDNS + Kademlia DHT peer discovery
│   │       ├── gossip.rs               # GossipSub for Polyp broadcast
│   │       ├── axon.rs                 # Axon: inbound request handler (serves queries)
│   │       ├── dendrite.rs             # Dendrite: outbound request sender (sends queries)
│   │       └── behaviour.rs            # Composed NetworkBehaviour
│   │
│   ├── chitin-consensus/               # Yuma-Semantic consensus, scoring, weights, bonds, epochs
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── yuma.rs                 # Yuma-Semantic consensus algorithm
│   │       ├── scoring.rs              # Multi-dimensional Polyp scoring
│   │       ├── weights.rs              # Weight matrix management, normalization
│   │       ├── bonds.rs                # Bond matrix, penalty computation
│   │       ├── epoch.rs                # Epoch lifecycle: open → score → commit → close
│   │       ├── metagraph.rs            # Metagraph state management
│   │       └── hardening.rs            # Hardening determination and CID anchoring
│   │
│   ├── chitin-reputation/              # Trust matrix, OpenRank, domain context, decay
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── trust_matrix.rs         # T_ij trust values per domain
│   │       ├── openrank.rs             # OpenRank integration for context-aware trust
│   │       ├── domain.rs               # Domain/topic classification for context scoping
│   │       └── decay.rs                # Time-decay functions for trust scores
│   │
│   ├── chitin-drift/                   # Drift detection, molting, alignment, versioning
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── detection.rs            # Semantic drift detection across model versions
│   │       ├── molting.rs              # Molting orchestration: re-embed + re-prove
│   │       ├── alignment.rs            # Cross-model vector space alignment (linear projection)
│   │       └── versioning.rs           # Model version registry and namespace management
│   │
│   ├── chitin-sync/                    # Vector Bloom Filters, set reconciliation, range sync
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── vbf.rs                  # Vector Bloom Filter construction and exchange
│   │       ├── reconcile.rs            # Set reconciliation: compute diff, request missing
│   │       └── range.rs                # Range-based sync for shard catchup
│   │
│   ├── chitin-economics/               # $CTN token, emission, staking, rewards, slashing, treasury
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── token.rs                # $CTN token type and supply tracking
│   │       ├── emission.rs             # Block reward emission schedule with halvings
│   │       ├── staking.rs              # Stake/unstake logic with cooldown periods
│   │       ├── rewards.rs              # Reward distribution: dividends + incentive
│   │       ├── slashing.rs             # Slashing conditions and penalty execution
│   │       └── treasury.rs             # Protocol treasury management
│   │
│   ├── chitin-rpc/                     # gRPC/JSON-RPC server + handlers
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── server.rs               # Tonic gRPC server setup
│   │       ├── handlers/
│   │       │   ├── polyp.rs            # Polyp CRUD + submission handlers
│   │       │   ├── query.rs            # Search and retrieval handlers
│   │       │   ├── node.rs             # Node info and health handlers
│   │       │   ├── wallet.rs           # Wallet and key management handlers
│   │       │   ├── staking.rs          # Stake/unstake handlers
│   │       │   ├── metagraph.rs        # Metagraph query handlers
│   │       │   ├── validation.rs       # Validation and scoring handlers
│   │       │   ├── sync.rs             # Sync status and trigger handlers
│   │       │   └── admin.rs            # Node admin and config handlers
│   │       └── middleware.rs           # Auth, rate limiting, logging
│   │
│   ├── chitin-daemon/                  # Coral/Tide node processes + scheduler + state
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs                 # Binary entrypoint
│   │       ├── coral.rs                # Coral Node: Polyp production pipeline
│   │       ├── tide.rs                 # Tide Node: validation and scoring pipeline
│   │       ├── scheduler.rs            # Epoch scheduler, periodic tasks
│   │       ├── state.rs                # Node state machine
│   │       └── config.rs              # Runtime configuration loading
│   │
│   └── chitin-cli/                     # Developer CLI (clap): init, wallet, polyp, query, etc.
│       ├── Cargo.toml
│       └── src/
│           ├── main.rs                 # CLI entrypoint (clap)
│           ├── commands/
│           │   ├── init.rs             # chitin init — scaffold node config
│           │   ├── wallet.rs           # chitin wallet — create/import/export keys
│           │   ├── polyp.rs            # chitin polyp — create/inspect/list polyps
│           │   ├── query.rs            # chitin query — semantic search
│           │   ├── stake.rs            # chitin stake — stake/unstake $CTN
│           │   ├── status.rs           # chitin status — node health and sync status
│           │   └── metagraph.rs        # chitin metagraph — view network state
│           └── output.rs              # Table/JSON output formatting
│
├── zk-circuits/                        # ZK proof circuits for Proof of Semantic Integrity
│   └── embedding-proof/
│       ├── Cargo.toml
│       └── src/
│           └── main.rs                 # zkVM guest: Vector = Model(Text) execution
│
├── sdk/
│   └── python/                         # chitin-py: Python SDK for AI Agents
│       ├── pyproject.toml
│       ├── chitin/
│       │   ├── __init__.py
│       │   ├── client.py               # gRPC/REST client to chitin-daemon
│       │   ├── agent.py                # ChitinMemory high-level agent interface
│       │   ├── types.py                # Python dataclasses mirroring Rust types
│       │   └── langchain.py            # LangChain VectorStore integration
│       └── tests/
│           └── test_client.py
│
├── protos/                             # Protocol Buffer definitions
│   ├── polyp.proto                     # Polyp message types
│   ├── query.proto                     # Search/retrieval messages
│   ├── consensus.proto                 # Validation and scoring messages
│   ├── metagraph.proto                 # Metagraph query messages
│   ├── economics.proto                 # Staking and reward messages
│   └── sync.proto                      # Synchronization messages
│
├── configs/
│   ├── model_configs.yaml              # Supported embedding model definitions
│   ├── consensus_params.yaml           # Yuma-Semantic consensus tuning parameters
│   └── economics.yaml                  # Token emission, staking, and slashing parameters
│
├── docker/
│   ├── Dockerfile                      # Multi-stage build for chitin-daemon
│   ├── Dockerfile.dev                  # Dev image with tooling
│   └── docker-compose.yml              # Full stack: Qdrant + IPFS + chitin-daemon
│
└── tests/
    ├── integration/                    # Cross-crate integration tests
    │   ├── consensus_test.rs
    │   ├── lifecycle_test.rs
    │   └── p2p_test.rs
    └── fixtures/                       # Test data
        ├── sample_polyps.json
        └── test_embeddings.bin
```

---

## 3. Core Data Structures

All types are defined in `chitin-core`. Downstream crates depend on these definitions.

### 3.1 Polyp and Lifecycle

```rust
// crates/chitin-core/src/polyp.rs

use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// Lifecycle states of a Polyp — from initial creation through consensus to hardening.
///
///   Draft ──► Soft ──► UnderReview ──► Approved ──► Hardened
///                          │                           ▲
///                          ▼                           │
///                       Rejected                    (immutable)
///                                                     │
///                                                   Molted (re-embedded with new model)
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub enum PolypState {
    /// Initial creation, not yet submitted to network.
    Draft,
    /// Submitted to network, ZK proof attached, awaiting validator pickup.
    Soft,
    /// Currently being evaluated by Tide Nodes in an active epoch.
    UnderReview,
    /// Passed Yuma-Semantic Consensus with sufficient validator agreement.
    Approved,
    /// Fully hardened: IPFS-pinned, CID anchored on-chain, attestations recorded.
    Hardened,
    /// Rejected by consensus — insufficient quality or failed ZK verification.
    Rejected,
    /// Superseded by a re-embedding under a newer model version (molting).
    Molted { successor_id: Uuid },
}

/// The atomic unit of knowledge in Reefipedia.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Polyp {
    /// Unique identifier (UUID v7 for time-ordering).
    pub id: Uuid,
    /// Current lifecycle state.
    pub state: PolypState,
    /// The knowledge content: text + embedding + provenance.
    pub subject: PolypSubject,
    /// ZK proof attesting Vector = Model(Text).
    pub proof: ZkProof,
    /// Consensus metadata (populated after validation).
    pub consensus: Option<ConsensusMetadata>,
    /// Hardening lineage (populated after hardening).
    pub hardening: Option<HardeningLineage>,
    /// Creation timestamp.
    pub created_at: DateTime<Utc>,
    /// Last state transition timestamp.
    pub updated_at: DateTime<Utc>,
}

/// The subject of a Polyp: payload (human-readable) + vector (machine-readable).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PolypSubject {
    /// Human-readable knowledge content.
    pub payload: Payload,
    /// Machine-readable vector embedding.
    pub vector: VectorEmbedding,
    /// Full provenance chain.
    pub provenance: Provenance,
}

/// The human-readable knowledge content.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Payload {
    /// The raw text, code snippet, or structured data.
    pub content: String,
    /// MIME type of the content (e.g., "text/plain", "text/markdown", "application/json").
    pub content_type: String,
    /// Optional: language code (e.g., "en", "es").
    pub language: Option<String>,
}

/// ZK proof attesting to correct embedding generation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ZkProof {
    /// Proof system identifier: "SP1Groth16", "Risc0Stark", etc.
    pub proof_type: String,
    /// Hex-encoded proof bytes.
    pub proof_value: String,
    /// The verification key hash (identifies the circuit).
    pub vk_hash: String,
    /// Public inputs committed in the proof.
    pub public_inputs: ProofPublicInputs,
    /// Timestamp of proof generation.
    pub created_at: DateTime<Utc>,
}

/// Public inputs committed inside the ZK proof.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProofPublicInputs {
    /// SHA-256 hash of the source text.
    pub text_hash: [u8; 32],
    /// SHA-256 hash of the resulting vector bytes.
    pub vector_hash: [u8; 32],
    /// Embedding model identifier.
    pub model_id: EmbeddingModelId,
}
```

### 3.2 Vector Embedding

```rust
// crates/chitin-core/src/embedding.rs

use serde::{Deserialize, Serialize};

/// Identifies a specific embedding model version.
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, Hash)]
pub struct EmbeddingModelId {
    /// Model family (e.g., "openai", "bge", "nomic").
    pub provider: String,
    /// Model name (e.g., "text-embedding-3-small").
    pub name: String,
    /// Exact version hash (SHA-256 of model weights).
    pub weights_hash: [u8; 32],
    /// Output dimensionality.
    pub dimensions: u32,
}

/// A vector embedding with full model provenance.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct VectorEmbedding {
    /// The raw float vector.
    pub values: Vec<f32>,
    /// Which model produced this vector.
    pub model_id: EmbeddingModelId,
    /// Quantization applied, if any (e.g., "float32", "int8", "binary").
    pub quantization: String,
    /// Normalization applied (e.g., "l2", "none").
    pub normalization: String,
}
```

### 3.3 Provenance

```rust
// crates/chitin-core/src/provenance.rs

use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// Full provenance chain for a Polyp.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Provenance {
    /// The agent/node that created this Polyp.
    pub creator: NodeIdentity,
    /// Attribution to the original source material.
    pub source: SourceAttribution,
    /// Processing pipeline that produced this Polyp.
    pub pipeline: ProcessingPipeline,
}

/// Attribution to the original source.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SourceAttribution {
    /// IPFS CID of the original source document (if available).
    pub source_cid: Option<String>,
    /// URL of the original source (if web-sourced).
    pub source_url: Option<String>,
    /// Human-readable title or description of the source.
    pub title: Option<String>,
    /// License under which the source is available.
    pub license: Option<String>,
    /// Timestamp when the source was accessed.
    pub accessed_at: DateTime<Utc>,
}

/// Describes the processing pipeline that produced the Polyp.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProcessingPipeline {
    /// Ordered list of processing steps (e.g., "chunk", "clean", "embed", "prove").
    pub steps: Vec<PipelineStep>,
    /// Total wall-clock time for the pipeline (milliseconds).
    pub duration_ms: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PipelineStep {
    pub name: String,
    pub version: String,
    pub params: serde_json::Value,
}
```

### 3.4 Node Identity

Follows Bittensor's coldkey/hotkey pattern for operational security.

```rust
// crates/chitin-core/src/identity.rs

use serde::{Deserialize, Serialize};

/// Identity of a node on the Chitin network.
///
/// Follows the coldkey/hotkey pattern:
/// - **Coldkey**: Long-term identity key, kept offline. Controls staking and rewards.
/// - **Hotkey**: Operational key, kept on the node. Signs messages and proofs.
///
/// The coldkey delegates authority to the hotkey. Compromising the hotkey
/// does not compromise staked funds (only the coldkey can unstake).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NodeIdentity {
    /// Coldkey public key (ed25519). Long-term identity.
    pub coldkey: [u8; 32],
    /// Hotkey public key (ed25519). Operational identity.
    pub hotkey: [u8; 32],
    /// DID derived from coldkey (e.g., "did:chitin:0xabc...").
    pub did: String,
    /// Node type.
    pub node_type: NodeType,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub enum NodeType {
    /// Produces Polyps.
    Coral,
    /// Validates and scores Polyps.
    Tide,
    /// Both producer and validator (allowed in Phase 1, restricted later).
    Hybrid,
}
```

### 3.5 Consensus and Scoring

```rust
// crates/chitin-core/src/consensus.rs

use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// Metadata attached to a Polyp after consensus evaluation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConsensusMetadata {
    /// Epoch in which this Polyp was evaluated.
    pub epoch: u64,
    /// Aggregate score after Yuma-Semantic Consensus.
    pub final_score: f64,
    /// Individual validator scores.
    pub validator_scores: Vec<ValidatorScore>,
    /// Whether this Polyp passed the hardening threshold.
    pub hardened: bool,
    /// Timestamp of consensus finalization.
    pub finalized_at: DateTime<Utc>,
}

/// A single validator's score for a Polyp.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ValidatorScore {
    /// The validator (Tide Node) that submitted this score.
    pub validator: [u8; 32],  // hotkey
    /// Multi-dimensional Polyp quality scores.
    pub scores: PolypScores,
    /// Validator's stake at time of scoring (for weight computation).
    pub stake_at_scoring: u64,
    /// Signature over the score payload.
    pub signature: Vec<u8>,
}

/// Multi-dimensional quality scores for a Polyp.
///
/// Each dimension is scored 0.0 to 1.0.
/// The final score is a weighted combination of all dimensions.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PolypScores {
    /// Did the ZK proof verify? Binary: 0.0 or 1.0.
    /// A score of 0.0 here causes immediate rejection regardless of other scores.
    pub zk_validity: f64,
    /// Semantic quality: coherence, informativeness, relevance.
    /// Evaluated by running the text through a quality classifier.
    pub semantic_quality: f64,
    /// Novelty: how much new information does this Polyp add?
    /// Measured by distance from nearest existing hardened Polyps.
    pub novelty: f64,
    /// Source credibility: reputation of the creator + quality of source attribution.
    pub source_credibility: f64,
    /// Embedding quality: cosine similarity between the vector and a reference
    /// embedding generated by the validator's own model instance.
    pub embedding_quality: f64,
}

impl PolypScores {
    /// Default dimension weights for computing final score.
    pub const DEFAULT_WEIGHTS: [f64; 5] = [0.30, 0.25, 0.15, 0.15, 0.15];

    /// Compute weighted final score.
    pub fn weighted_score(&self) -> f64 {
        let vals = [
            self.zk_validity,
            self.semantic_quality,
            self.novelty,
            self.source_credibility,
            self.embedding_quality,
        ];
        vals.iter()
            .zip(Self::DEFAULT_WEIGHTS.iter())
            .map(|(v, w)| v * w)
            .sum()
    }
}

/// Validator attestation for a hardened Polyp.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Attestation {
    /// Validator hotkey that signed.
    pub validator: [u8; 32],
    /// Epoch of attestation.
    pub epoch: u64,
    /// The Polyp ID being attested.
    pub polyp_id: Uuid,
    /// The CID of the hardened Polyp on IPFS.
    pub cid: String,
    /// ed25519 signature over (polyp_id || cid || epoch).
    pub signature: Vec<u8>,
}

/// Lineage information for a hardened Polyp.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HardeningLineage {
    /// IPFS CID of the hardened Polyp.
    pub cid: String,
    /// Merkle proof linking this Polyp to the epoch Merkle root.
    pub merkle_proof: Vec<[u8; 32]>,
    /// Epoch Merkle root.
    pub merkle_root: [u8; 32],
    /// Validator attestations.
    pub attestations: Vec<Attestation>,
    /// On-chain transaction hash anchoring the Merkle root (if applicable).
    pub anchor_tx: Option<String>,
    /// Timestamp of hardening.
    pub hardened_at: DateTime<Utc>,
}
```

### 3.6 Reef Metagraph

```rust
// crates/chitin-core/src/metagraph.rs

use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// The global network state — all nodes, their stakes, trust, and performance.
///
/// Analogous to Bittensor's Metagraph. Updated every epoch.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReefMetagraph {
    /// Current epoch number.
    pub epoch: u64,
    /// Block height (if anchored to a chain).
    pub block: u64,
    /// All registered nodes.
    pub nodes: Vec<NodeInfo>,
    /// Total staked $CTN across all nodes.
    pub total_stake: u64,
    /// Total Polyps hardened to date.
    pub total_hardened_polyps: u64,
    /// Current emission rate (tokens per epoch).
    pub emission_rate: u64,
    /// Weight matrix W[validator_uid][coral_uid] = weight.
    /// Sparse representation: only non-zero entries.
    pub weights: HashMap<u16, Vec<(u16, f64)>>,
    /// Bond matrix B[validator_uid][coral_uid] = bond.
    pub bonds: HashMap<u16, Vec<(u16, f64)>>,
}

/// Information about a single node in the metagraph.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NodeInfo {
    /// Network-assigned unique ID (0..n).
    pub uid: u16,
    /// Hotkey (operational identity).
    pub hotkey: [u8; 32],
    /// Coldkey (staking identity).
    pub coldkey: [u8; 32],
    /// Node type.
    pub node_type: NodeType,
    /// Total $CTN staked (own + delegated).
    pub stake: u64,
    /// Trust score (EMA of validation performance).
    pub trust: f64,
    /// Consensus score (agreement with other validators).
    pub consensus: f64,
    /// Incentive score (share of epoch rewards).
    pub incentive: f64,
    /// Emission received this epoch.
    pub emission: u64,
    /// Number of Polyps produced (Coral) or validated (Tide).
    pub polyp_count: u64,
    /// Last active epoch.
    pub last_active: u64,
    /// Network address (multiaddr).
    pub axon_addr: String,
    /// Whether currently registered and active.
    pub active: bool,
}
```

---

## 4. Yuma-Semantic Consensus Algorithm

The Yuma-Semantic Consensus adapts Bittensor's Yuma Consensus for semantic knowledge validation. Instead of evaluating model inference quality, it evaluates Polyp quality across five dimensions.

### 4.1 Overview

Each epoch (configurable, default 360 blocks / ~1 hour):

1. Tide Nodes request Polyps from Coral Nodes via `ValidationQuery`.
2. Tide Nodes evaluate each Polyp and produce `PolypScores`.
3. Tide Nodes submit weight vectors: `W[validator][coral] = aggregated_score`.
4. The consensus algorithm runs to determine:
   - Which Coral Nodes produced high-quality Polyps (incentive).
   - Which Tide Nodes scored accurately (dividends).
   - Which Polyps should be hardened.

### 4.2 Algorithm Pseudocode

```
FUNCTION yuma_semantic_consensus(
    S: Vec<u64>,               // stake per validator
    W: Matrix<f64>,            // weight matrix [validators × corals]
    B_prev: Matrix<f64>,       // previous bond matrix
    kappa: f64,                // consensus threshold (default 0.5)
    bond_penalty: f64,         // bond decay rate (default 0.1)
    alpha: f64,                // EMA smoothing factor (default 0.1)
) -> ConsensusResult:

    n_validators = len(S)
    n_corals = W.cols()

    // ──────────────────────────────────────────────
    // STEP 1: Stake Normalization
    // ──────────────────────────────────────────────
    // Normalize validator stakes to sum to 1.0.
    total_stake = sum(S)
    s_norm[i] = S[i] / total_stake   for i in 0..n_validators

    // ──────────────────────────────────────────────
    // STEP 2: Weight Clipping
    // ──────────────────────────────────────────────
    // Clip each validator's weights to prevent outsized influence.
    // No single validator can assign more than `max_weight_fraction` to one coral.
    max_weight_fraction = 0.5
    FOR each validator i:
        row_sum = sum(W[i][*])
        IF row_sum > 0:
            W[i][j] = W[i][j] / row_sum           for all j   // normalize to sum=1
            W[i][j] = min(W[i][j], max_weight_fraction) for all j
            // re-normalize after clipping
            row_sum2 = sum(W[i][*])
            W[i][j] = W[i][j] / row_sum2           for all j

    // ──────────────────────────────────────────────
    // STEP 3: Stake-Weighted Median (per Coral Node)
    // ──────────────────────────────────────────────
    // For each Coral Node j, compute the stake-weighted median of all
    // validator scores. This is the "consensus score" for that Coral.
    FOR each coral j:
        // Collect (weight, stake) pairs, sorted by weight ascending
        pairs = sorted([(W[i][j], s_norm[i]) for i in 0..n_validators], by=weight)
        // Walk through sorted pairs accumulating stake until we pass kappa
        cumulative_stake = 0.0
        consensus_weight[j] = 0.0
        FOR (weight, stake) in pairs:
            cumulative_stake += stake
            IF cumulative_stake >= kappa:
                consensus_weight[j] = weight
                BREAK

    // ──────────────────────────────────────────────
    // STEP 4: Validator Agreement (Consensus Score)
    // ──────────────────────────────────────────────
    // Measure how closely each validator's weights align with consensus.
    // Validators that agree with the majority earn higher dividends.
    FOR each validator i:
        agreement[i] = 0.0
        FOR each coral j:
            // Agreement = 1 - |deviation from consensus|
            deviation = abs(W[i][j] - consensus_weight[j])
            agreement[i] += (1.0 - deviation) * s_norm[i]
        // Normalize
        agreement[i] = agreement[i] / n_corals

    // ──────────────────────────────────────────────
    // STEP 5: Bond Matrix Update
    // ──────────────────────────────────────────────
    // Bonds represent a validator's historical commitment to scoring a coral.
    // EMA update: B[i][j] = alpha * W[i][j] + (1-alpha) * B_prev[i][j]
    // Penalty: if validator disagreed with consensus, bonds decay faster.
    FOR each validator i:
        FOR each coral j:
            ema = alpha * W[i][j] + (1.0 - alpha) * B_prev[i][j]
            // Apply penalty if validator deviated from consensus
            penalty = bond_penalty * abs(W[i][j] - consensus_weight[j])
            B[i][j] = max(0.0, ema - penalty)

    // ──────────────────────────────────────────────
    // STEP 6: Incentive Computation
    // ──────────────────────────────────────────────
    // Coral incentive: proportional to consensus weight (quality of Polyps).
    total_cw = sum(consensus_weight[*])
    IF total_cw > 0:
        incentive[j] = consensus_weight[j] / total_cw  for each coral j
    ELSE:
        incentive[j] = 0.0

    // Validator dividends: proportional to agreement * stake * bonds.
    FOR each validator i:
        bond_sum = sum(B[i][j] * consensus_weight[j] for j in 0..n_corals)
        dividend[i] = agreement[i] * s_norm[i] * bond_sum

    // Normalize dividends
    total_div = sum(dividend[*])
    IF total_div > 0:
        dividend[i] = dividend[i] / total_div  for each validator i

    // ──────────────────────────────────────────────
    // STEP 7: Hardening Determination
    // ──────────────────────────────────────────────
    // A Polyp is hardened if:
    //   1. Its consensus_weight exceeds the hardening threshold.
    //   2. At least `min_attestations` validators scored it above threshold.
    //   3. ZK validity was confirmed by all scoring validators.
    hardening_threshold = 0.6
    min_attestations = 3

    FOR each coral j:
        polyps_from_j = get_epoch_polyps(coral=j)
        FOR each polyp p in polyps_from_j:
            attestation_count = count(
                validators where PolypScores.zk_validity == 1.0
                AND PolypScores.weighted_score() >= hardening_threshold
            )
            IF consensus_weight[j] >= hardening_threshold
               AND attestation_count >= min_attestations:
                harden(p)   // IPFS pin + CID anchor + Merkle proof

    RETURN ConsensusResult {
        consensus_weights,
        incentives,
        dividends,
        bonds: B,
        hardened_polyps,
    }
```

### 4.3 EMA Scoring

Tide Nodes maintain running EMA scores for each Coral Node they interact with. This smooths out epoch-to-epoch variance and rewards consistent quality.

```
// Per-validator EMA for each Coral Node
scores[coral_uid] = alpha * new_epoch_score + (1.0 - alpha) * scores[coral_uid]

// Default alpha = 0.1 (slow adaptation, stable scores)
// Higher alpha = faster adaptation to quality changes but more noise
```

### 4.4 Anti-Sybil Measures

1. **Stake requirement**: Both Coral and Tide Nodes must stake $CTN to participate. Minimum stake is configurable per Reef Zone.
2. **Weight clipping**: No single validator can dominate scoring (max 50% weight to any coral).
3. **Bond penalty**: Validators that consistently disagree with consensus see their bonds decay, reducing their dividend share over time.
4. **Proof-of-Unique-Work**: Each Polyp's ZK proof binds it to a specific (text, model, vector) tuple. You cannot reuse proofs across different texts.
5. **Novelty scoring**: Polyps that are near-duplicates of existing hardened Polyps receive low novelty scores, discouraging spam.

---

## 5. Data Flow Diagrams

### 5.1 Complete Polyp Lifecycle

```
                          POLYP LIFECYCLE
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  ┌──────────┐    ┌──────────┐    ┌────────────┐                 │
  │  │  Source  │───►│  Chunk   │───►│  Embed in  │                 │
  │  │  Text    │    │  + Clean │    │  zkVM      │                 │
  │  └──────────┘    └──────────┘    └─────┬──────┘                 │
  │                                        │                        │
  │                                        ▼                        │
  │                                 ┌──────────────┐                │
  │                                 │  ZK Proof    │                │
  │                                 │  Generated   │                │
  │                                 └──────┬───────┘                │
  │                                        │                        │
  │                                        ▼                        │
  │  ┌─────────────────────────────────────────────────────────┐    │
  │  │                   CORAL NODE                            │    │
  │  │  Assembles Polyp: payload + vector + proof + provenance │    │
  │  │  State: Draft ──► Soft                                  │    │
  │  └──────────────────────────┬──────────────────────────────┘    │
  │                             │                                   │
  │                             │ PolypSubmission (gossip)          │
  │                             ▼                                   │
  │  ┌─────────────────────────────────────────────────────────┐    │
  │  │                   TIDE NODES (multiple)                 │    │
  │  │                                                         │    │
  │  │  1. Receive Polyp via ValidationQuery                   │    │
  │  │  2. Verify ZK proof (constant-time)                     │    │
  │  │  3. Evaluate 5 scoring dimensions                       │    │
  │  │  4. Submit PolypScores                                  │    │
  │  │  State: Soft ──► UnderReview                            │    │
  │  └──────────────────────────┬──────────────────────────────┘    │
  │                             │                                   │
  │                             │ ValidationScoreSubmission         │
  │                             ▼                                   │
  │  ┌─────────────────────────────────────────────────────────┐    │
  │  │              YUMA-SEMANTIC CONSENSUS                    │    │
  │  │                                                         │    │
  │  │  • Stake-weighted median scoring                        │    │
  │  │  • Weight clipping + bonds penalty                      │    │
  │  │  • Incentive + dividend computation                     │    │
  │  │  • Hardening determination                              │    │
  │  │  State: UnderReview ──► Approved OR Rejected            │    │
  │  └──────────────────────────┬──────────────────────────────┘    │
  │                             │                                   │
  │                     ┌───────┴───────┐                           │
  │                     │   Approved?   │                           │
  │                     └───┬───────┬───┘                           │
  │                    yes  │       │  no                           │
  │                         ▼       ▼                               │
  │  ┌──────────────────────┐  ┌──────────┐                         │
  │  │  CHITIN HARDENING    │  │ Rejected │                         │
  │  │                      │  │ (logged) │                         │
  │  │  • IPFS pin          │  └──────────┘                         │
  │  │  • CID anchor        │                                       │
  │  │  • Merkle proof      │                                       │
  │  │  • Attestations      │                                       │
  │  │  State: ──► Hardened │                                       │
  │  └──────────┬───────────┘                                       │
  │             │                                                   │
  │             ▼                                                   │
  │  ┌──────────────────────┐                                       │
  │  │  THE REEF            │                                       │
  │  │  (queryable via ANN) │◄──── RetrievalRequest ──── Agents     │
  │  └──────────────────────┘                                       │
  │                                                                 │
  └─────────────────────────────────────────────────────────────────┘
```

### 5.2 Coral Node Internal Architecture

```
  ┌──────────────────────── CORAL NODE ────────────────────────┐
  │                                                            │
  │  ┌──────────┐    ┌───────────┐    ┌──────────────────┐     │
  │  │ Ingest   │───►│ Pipeline  │───►│ zkVM Prover      │     │
  │  │ Queue    │    │ Manager   │    │ (SP1 / Risc0)    │     │
  │  └──────────┘    └───────────┘    └────────┬─────────┘     │
  │       ▲                                    │               │
  │       │                                    ▼               │
  │  ┌──────────┐                      ┌──────────────────┐    │
  │  │ RPC API  │                      │ Polyp Assembler  │    │
  │  │ (submit) │                      │ (payload+vec+    │    │
  │  └──────────┘                      │  proof+prov)     │    │
  │       ▲                            └────────┬─────────┘    │
  │       │                                     │              │
  │  ┌──────────┐                               ▼              │
  │  │ Python   │                      ┌──────────────────┐    │
  │  │ SDK /    │                      │ P2P Gossip       │    │
  │  │ CLI      │                      │ (broadcast Soft  │    │
  │  └──────────┘                      │  Polyp)          │    │
  │                                    └──────────────────┘    │
  │                                                            │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │ Local Store (RocksDB)                                │  │
  │  │ • Draft/Soft Polyps awaiting consensus               │  │
  │  │ • Model weights cache                                │  │
  │  │ • Proof cache                                        │  │
  │  └──────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────┘
```

### 5.3 Tide Node Internal Architecture

```
  ┌──────────────────────── TIDE NODE ────────────────────────┐
  │                                                           │
  │  ┌──────────────────┐    ┌────────────────────────────┐   │
  │  │ Epoch Scheduler  │───►│ ValidationQuery Sender     │   │
  │  │ (triggers at     │    │ (dendrite → coral axons)   │   │
  │  │  epoch boundary) │    └───────────┬────────────────┘   │
  │  └──────────────────┘                │                    │
  │                                      ▼                    │
  │                             ┌──────────────────┐          │
  │                             │ Polyp Receiver   │          │
  │                             │ (collect from    │          │
  │                             │  multiple corals)│          │
  │                             └────────┬─────────┘          │
  │                                      │                    │
  │                         ┌────────────┼────────────┐       │
  │                         ▼            ▼            ▼       │
  │                  ┌───────────┐ ┌──────────┐ ┌──────────┐  │
  │                  │ ZK Verify │ │ Semantic │ │ Novelty  │  │
  │                  │ (proof    │ │ Quality  │ │ Checker  │  │
  │                  │  check)   │ │ Scorer   │ │ (ANN     │  │
  │                  └─────┬─────┘ └────┬─────┘ │ lookup)  │  │
  │                        │            │       └────┬─────┘  │
  │                        └────────────┼────────────┘        │
  │                                     ▼                     │
  │                          ┌───────────────────┐            │
  │                          │ Score Aggregator  │            │
  │                          │ (PolypScores →    │            │
  │                          │  EMA update →     │            │
  │                          │  weight vector)   │            │
  │                          └────────┬──────────┘            │
  │                                   │                       │
  │                                   ▼                       │
  │                          ┌───────────────────┐            │
  │                          │ Weight Submitter  │            │
  │                          │ (set_weights per  │            │
  │                          │  epoch)           │            │
  │                          └───────────────────┘            │
  │                                                           │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │ Local Store (RocksDB)                               │  │
  │  │ • EMA scores per coral                              │  │
  │  │ • Bond history                                      │  │
  │  │ • Local HNSW index (for novelty checking)           │  │
  │  │ • Hardened Polyp cache                              │  │
  │  └─────────────────────────────────────────────────────┘  │
  └───────────────────────────────────────────────────────────┘
```

---

## 6. Message Protocol

Six message types govern all inter-node communication. Serialized as Protocol Buffers over libp2p streams.

### 6.1 PolypSubmission

Coral Node → Network (gossip broadcast).

```
message PolypSubmission {
    bytes   polyp_id      = 1;   // UUID bytes
    bytes   hotkey        = 2;   // Coral's hotkey
    Payload payload       = 3;   // Text content
    Vector  vector        = 4;   // Embedding
    ZkProof proof         = 5;   // ZK attestation
    Provenance provenance = 6;   // Source chain
    bytes   signature     = 7;   // ed25519 over hash(payload+vector+proof)
    uint64  timestamp     = 8;   // Unix millis
}
```

### 6.2 ValidationQuery

Tide Node → Coral Node (direct request via dendrite→axon).

```
message ValidationQuery {
    bytes  validator_hotkey = 1;  // Tide Node requesting
    uint64 epoch            = 2;  // Current epoch
    uint32 max_polyps       = 3;  // Max polyps to return (default 32)
    string reef_zone        = 4;  // Topic filter (e.g., "code/python")
    bytes  signature        = 5;  // Auth signature
}
```

### 6.3 ValidationResponse

Coral Node → Tide Node (response to ValidationQuery).

```
message ValidationResponse {
    bytes             coral_hotkey = 1;
    uint64            epoch        = 2;
    repeated PolypSubmission polyps = 3;  // Batch of polyps for evaluation
    bytes             signature    = 4;
}
```

### 6.4 ValidationScoreSubmission

Tide Node → Consensus Layer (epoch-end submission).

```
message ValidationScoreSubmission {
    bytes  validator_hotkey = 1;
    uint64 epoch            = 2;

    // Sparse weight vector: (coral_uid, weight) pairs
    repeated WeightEntry weights = 3;

    // Per-polyp detailed scores (optional, for auditing)
    repeated PolypScoreEntry detailed_scores = 4;

    bytes  signature = 5;
}

message WeightEntry {
    uint32 coral_uid = 1;
    double weight    = 2;
}

message PolypScoreEntry {
    bytes  polyp_id          = 1;
    double zk_validity       = 2;
    double semantic_quality  = 3;
    double novelty           = 4;
    double source_credibility = 5;
    double embedding_quality = 6;
}
```

### 6.5 HardeningReceipt

Consensus Layer → Network (broadcast after hardening).

```
message HardeningReceipt {
    bytes              polyp_id     = 1;
    string             cid          = 2;  // IPFS CID
    bytes              merkle_root  = 3;  // Epoch merkle root
    repeated bytes     merkle_proof = 4;  // Proof of inclusion
    repeated Attestation attestations = 5;
    uint64             epoch        = 6;
    string             anchor_tx    = 7;  // On-chain tx hash (optional)
}

message Attestation {
    bytes  validator_hotkey = 1;
    uint64 epoch            = 2;
    bytes  polyp_id         = 3;
    string cid              = 4;
    bytes  signature        = 5;
}
```

### 6.6 RetrievalRequest / RetrievalResponse

Agent → Any Node (query for knowledge).

```
message RetrievalRequest {
    string query_text     = 1;  // Natural language query
    Vector query_vector   = 2;  // Pre-computed query vector (optional)
    string model_id       = 3;  // Which embedding space to search
    uint32 top_k          = 4;  // Number of results (default 10)
    double min_trust      = 5;  // Minimum trust score filter (default 0.0)
    bool   hardened_only  = 6;  // Only return hardened polyps (default true)
    string reef_zone      = 7;  // Topic filter (optional)
}

message RetrievalResponse {
    repeated RetrievalResult results = 1;
    uint64 search_time_ms           = 2;
    string served_by                = 3;  // Node DID that served the query
}

message RetrievalResult {
    bytes    polyp_id        = 1;
    string   cid             = 2;
    Payload  payload         = 3;
    double   similarity      = 4;  // Cosine similarity to query
    double   trust_score     = 5;  // Composite trust score
    string   creator_did     = 6;
    string   state           = 7;  // "Hardened", "Approved", etc.
    uint64   hardened_epoch  = 8;
}
```

---

## 7. Incentive Layer

### 7.1 $CTN (Chitin) Token

| Parameter | Value |
|---|---|
| Token name | Chitin |
| Symbol | $CTN |
| Maximum supply | 21,000,000 CTN |
| Decimal precision | 9 (1 CTN = 10^9 rao) |
| Initial block reward | 1.0 CTN per block |
| Block time | ~12 seconds |
| Halving interval | Every 10,512,000 blocks (~4 years) |
| Halving schedule | 1.0 → 0.5 → 0.25 → 0.125 → ... |

### 7.2 Emission Schedule

```
Block Reward (epoch e):
    halving_number = floor(total_blocks / 10_512_000)
    reward = 1.0 / (2 ^ halving_number)

Epoch Emission:
    E_epoch = blocks_per_epoch * reward
```

Epoch emission is split between Coral Nodes and Tide Nodes:

```
E = E_epoch

Coral share:   E_coral = (1 - xi) * E    // xi = validator_fraction, default 0.41
Tide share:    E_tide  = xi * E

// Distribution within each share:
Coral Node j receives: E_coral * incentive[j]
Tide Node i receives:  E_tide  * dividend[i]
```

The reward formula:

```
Reward for node n:
    R(n) = xi * D(n) + (1 - xi) * I(n)

Where:
    D(n) = dividend share (validators only, proportional to agreement * bonds)
    I(n) = incentive share (corals only, proportional to consensus weight)
    xi   = validator emission fraction (default 0.41)
```

### 7.3 Staking Requirements

| Node Type | Minimum Stake | Cooldown (unstaking) |
|---|---|---|
| Coral Node | 100 CTN | 7,200 blocks (~24 hours) |
| Tide Node | 1,000 CTN | 21,600 blocks (~72 hours) |
| Delegation | 10 CTN | 7,200 blocks (~24 hours) |

Delegation: any token holder can delegate stake to a Coral or Tide Node. The delegator receives a proportional share of the node's rewards minus a configurable commission rate (default 18%).

### 7.4 Slashing Conditions

Four conditions trigger slashing (partial or full stake forfeiture):

| Condition | Severity | Penalty | Description |
|---|---|---|---|
| **Invalid ZK Proof** | Critical | 100% of stake | Coral Node submitted a Polyp with a ZK proof that does not verify. Indicates dishonest embedding generation. |
| **Consensus Deviation** | Moderate | 5% of stake per offense | Tide Node consistently scores in strong disagreement with consensus (>3 sigma deviation for 3+ consecutive epochs). Indicates collusion or incompetence. |
| **Liveness Failure** | Low | 1% of stake per missed epoch | Node fails to respond to ValidationQueries or submit weights for 3+ consecutive epochs. |
| **Duplicate Submission** | Moderate | 10% of stake | Coral Node submits a Polyp that is a near-duplicate (cosine similarity > 0.98) of an existing hardened Polyp in the same model namespace. |

Slashed tokens flow to the protocol treasury (see Section 7.5).

### 7.5 Treasury

- **Treasury allocation**: 2% of each epoch's emission goes to the protocol treasury.
- **Slash proceeds**: All slashed tokens go to the treasury.
- **Governance**: Treasury spending is governed by $CTN token-weighted voting (Phase 3+).
- **Uses**: Fund development, security audits, migration incentives (molting), ecosystem grants.

---

## 8. Configuration Schemas

### 8.1 model_configs.yaml

Defines supported embedding models and their properties.

```yaml
# configs/model_configs.yaml
models:
  - id: "openai/text-embedding-3-small"
    provider: "openai"
    name: "text-embedding-3-small"
    dimensions: 1536
    quantization: "float32"
    normalization: "l2"
    weights_hash: "sha256:a1b2c3d4..."
    max_tokens: 8191
    zkvm_compatible: true
    zkvm_target: "sp1"
    status: "active"

  - id: "bge/bge-small-en-v1.5"
    provider: "bge"
    name: "bge-small-en-v1.5"
    dimensions: 384
    quantization: "float32"
    normalization: "l2"
    weights_hash: "sha256:e5f6g7h8..."
    max_tokens: 512
    zkvm_compatible: true
    zkvm_target: "sp1"
    status: "active"

  - id: "nomic/nomic-embed-text-v1.5"
    provider: "nomic"
    name: "nomic-embed-text-v1.5"
    dimensions: 768
    quantization: "float32"
    normalization: "l2"
    weights_hash: "sha256:i9j0k1l2..."
    max_tokens: 8192
    zkvm_compatible: true
    zkvm_target: "risc0"
    status: "active"

# Default model for new Polyps
default_model: "bge/bge-small-en-v1.5"

# Alignment projections between model spaces
alignments:
  - from: "bge/bge-small-en-v1.5"
    to: "openai/text-embedding-3-small"
    projection_cid: "bafybeig..."  # IPFS CID of the projection matrix
    accuracy_loss: 0.03            # Expected recall degradation
```

### 8.2 consensus_params.yaml

Tuning parameters for Yuma-Semantic Consensus.

```yaml
# configs/consensus_params.yaml
consensus:
  # Epoch duration in blocks
  epoch_length: 360               # ~1 hour at 10s/block

  # Yuma consensus parameters
  kappa: 0.5                      # Stake-weighted median threshold
  bond_penalty: 0.1               # Bond decay rate for disagreeing validators
  alpha: 0.1                      # EMA smoothing factor

  # Weight constraints
  max_weight_fraction: 0.5        # Max weight one validator can assign to one coral
  min_validators_per_polyp: 3     # Minimum attestations for hardening

  # Hardening thresholds
  hardening_threshold: 0.6        # Minimum consensus weight to harden
  zk_validity_required: true      # ZK proof must verify for hardening

  # Polyp scoring dimension weights (must sum to 1.0)
  score_weights:
    zk_validity: 0.30
    semantic_quality: 0.25
    novelty: 0.15
    source_credibility: 0.15
    embedding_quality: 0.15

  # Anti-Sybil parameters
  novelty_duplicate_threshold: 0.98  # Cosine sim above this = duplicate
  max_polyps_per_coral_per_epoch: 64 # Rate limit per coral per epoch

  # Metagraph update frequency
  metagraph_sync_interval: 100    # Blocks between metagraph snapshots
```

### 8.3 economics.yaml

Token economics and staking parameters.

```yaml
# configs/economics.yaml
token:
  name: "Chitin"
  symbol: "CTN"
  max_supply: 21_000_000
  decimals: 9                     # 1 CTN = 1_000_000_000 rao
  block_time_seconds: 12

emission:
  initial_block_reward: 1.0       # CTN per block
  halving_interval: 10_512_000    # Blocks (~4 years)
  treasury_fraction: 0.02         # 2% of emission to treasury
  validator_fraction: 0.41        # xi: fraction of emission to validators

staking:
  coral_minimum: 100              # CTN
  tide_minimum: 1_000             # CTN
  delegation_minimum: 10          # CTN
  coral_cooldown_blocks: 7_200    # ~24 hours
  tide_cooldown_blocks: 21_600    # ~72 hours
  delegation_cooldown_blocks: 7_200
  default_commission_rate: 0.18   # 18% delegation commission

slashing:
  invalid_zk_proof: 1.0           # 100% of stake
  consensus_deviation: 0.05       # 5% per offense
  consensus_deviation_sigma: 3.0  # Standard deviations for trigger
  consensus_deviation_epochs: 3   # Consecutive epochs before slash
  liveness_failure: 0.01          # 1% per missed epoch
  liveness_failure_epochs: 3      # Consecutive misses before slash
  duplicate_submission: 0.10      # 10% of stake
```

---

## 9. Phase-Gated Feature Matrix

Features are gated across three deployment phases. Each phase builds on the previous.

| Feature | Phase 1: Local MVP | Phase 2: Federation | Phase 3: Full Chitin |
|---|---|---|---|
| **Polyp creation** | Local only | P2P broadcast | P2P + consensus |
| **ZK proofs** | Optional (dev mode) | Required for federation | Required |
| **Storage: IPFS** | Local Kubo node | Federated pinning | Incentivized pinning market |
| **Storage: Vector index** | Local Qdrant | Sharded across peers | Distributed HNSW with consensus |
| **Consensus** | None (TOFU) | Signed Polyps + local EigenTrust | Full Yuma-Semantic |
| **Node types** | Hybrid only | Coral + Tide (soft separation) | Strict Coral / Tide |
| **Networking** | Localhost RPC | libp2p mesh (permissioned) | libp2p mesh (permissionless) |
| **Identity** | Local keypair | Coldkey/hotkey | Coldkey/hotkey + DID |
| **Reputation** | None | Local trust matrix | OpenRank + domain context |
| **Token economics** | None | Reputation points (non-transferable) | $CTN token (full tokenomics) |
| **Staking** | None | None | Required for participation |
| **Slashing** | None | Peer banning | On-chain slashing |
| **Molting** | Manual re-embed | Coordinated re-embed | Incentivized migration agents |
| **Sync** | N/A | GossipSub | Vector Bloom Filters + range sync |
| **API** | Full local RPC | Full local + federated query | Full local + federated + metagraph |
| **CLI** | init, polyp, query | + wallet, status | + stake, metagraph |
| **Python SDK** | remember / recall | + trust filtering | + staking, delegation |
| **Docker** | Single container | Multi-node compose | Helm chart / k8s |
| **Metagraph** | None | Local view | Full Reef Metagraph |
| **Hardening** | Instant (no consensus) | Quorum-signed | Full Chitin Hardening |
| **Reef Zones** | Single namespace | Topic-based subscriptions | Incentivized zones |

---

## 10. API Endpoints

All endpoints are served by `chitin-rpc` as gRPC services with optional JSON-RPC/REST gateway.

### 10.1 Polyp Management

| Method | Endpoint | Description |
|---|---|---|
| `SubmitPolyp` | `/polyp/submit` | Submit a new Polyp (text + vector + proof). Returns Polyp ID. |
| `GetPolyp` | `/polyp/get` | Retrieve a Polyp by ID. |
| `ListPolyps` | `/polyp/list` | List Polyps with filters (state, creator, epoch, model). |
| `GetPolypState` | `/polyp/state` | Get current lifecycle state of a Polyp. |
| `GetPolypProvenance` | `/polyp/provenance` | Get full provenance chain for a Polyp. |
| `GetHardeningReceipt` | `/polyp/hardening` | Get hardening receipt (CID, Merkle proof, attestations). |

### 10.2 Query / Retrieval

| Method | Endpoint | Description |
|---|---|---|
| `SemanticSearch` | `/query/search` | ANN search: query text or vector → top-k results. |
| `HybridSearch` | `/query/hybrid` | Combined semantic + keyword search. |
| `GetByCid` | `/query/cid` | Retrieve a hardened Polyp directly by IPFS CID. |
| `ExplainResult` | `/query/explain` | Explain why a result matched (similarity breakdown). |

### 10.3 Node

| Method | Endpoint | Description |
|---|---|---|
| `GetNodeInfo` | `/node/info` | Node type, version, uptime, capabilities. |
| `GetHealth` | `/node/health` | Health check: storage, P2P, index status. |
| `GetPeers` | `/node/peers` | List connected peers. |

### 10.4 Wallet

| Method | Endpoint | Description |
|---|---|---|
| `CreateWallet` | `/wallet/create` | Generate new coldkey/hotkey pair. |
| `ImportWallet` | `/wallet/import` | Import existing keys. |
| `GetBalance` | `/wallet/balance` | Get $CTN balance for a coldkey. |
| `Transfer` | `/wallet/transfer` | Transfer $CTN between coldkeys. |

### 10.5 Staking

| Method | Endpoint | Description |
|---|---|---|
| `Stake` | `/staking/stake` | Stake $CTN to a node (own or delegation). |
| `Unstake` | `/staking/unstake` | Begin unstaking (starts cooldown). |
| `GetStakeInfo` | `/staking/info` | Get staking info: amount, delegators, cooldown status. |

### 10.6 Metagraph

| Method | Endpoint | Description |
|---|---|---|
| `GetMetagraph` | `/metagraph/get` | Full metagraph snapshot for current epoch. |
| `GetNodeMetrics` | `/metagraph/node` | Metrics for a specific node (trust, consensus, incentive). |
| `GetWeights` | `/metagraph/weights` | Weight matrix for current epoch. |
| `GetBonds` | `/metagraph/bonds` | Bond matrix for current epoch. |

### 10.7 Validation

| Method | Endpoint | Description |
|---|---|---|
| `SubmitScores` | `/validation/scores` | Tide Node submits epoch scores/weights. |
| `GetEpochStatus` | `/validation/epoch` | Current epoch info: number, phase, time remaining. |
| `GetConsensusResult` | `/validation/result` | Consensus result for a completed epoch. |

### 10.8 Sync

| Method | Endpoint | Description |
|---|---|---|
| `GetSyncStatus` | `/sync/status` | Sync progress: blocks behind, peers syncing from. |
| `TriggerSync` | `/sync/trigger` | Manually trigger sync with peers. |

### 10.9 Admin

| Method | Endpoint | Description |
|---|---|---|
| `GetConfig` | `/admin/config` | Get current node configuration. |
| `UpdateConfig` | `/admin/config/update` | Hot-reload configuration parameters. |
| `GetLogs` | `/admin/logs` | Stream node logs. |

---

## 11. Consciousness Repository Relationship

The Consciousness Repository contains a 40-form architecture — a structured framework of philosophical, cognitive, and epistemic forms that represent different modes of understanding and knowledge representation.

### How the 40-Form Architecture Feeds Reefipedia

1. **Initial Seed Content**: Each of the 40 forms can be decomposed into canonical knowledge chunks that become the first Polyps in the Reef. These serve as the "bedrock layer" — high-quality, well-sourced foundational knowledge.

2. **Reef Zone Templates**: The 40 forms map naturally to Reef Zones (topic partitions). For example:
   - Epistemic forms → "epistemology" zone
   - Cognitive forms → "cognitive-science" zone
   - Ethical forms → "ethics" zone

3. **Quality Benchmarks**: The manually curated 40-form Polyps serve as reference points for semantic quality scoring. Tide Nodes can compare new submissions against these benchmarks when computing the `semantic_quality` dimension of PolypScores.

4. **Embedding Model Calibration**: The 40-form corpus provides a stable test set for evaluating embedding model quality. When a new model is considered for addition to `model_configs.yaml`, its performance on the 40-form Polyps serves as a baseline.

5. **Molting Reference Set**: During model migrations (molting), the 40-form Polyps are always migrated first. Their known semantic relationships serve as ground truth to validate that the new model's vector space preserves critical knowledge structure.

### Seeding Pipeline

```
consciousness/40-forms/
  ├── form_01_phenomenal.md     ──► chunk ──► embed ──► prove ──► Polyp
  ├── form_02_reflexive.md      ──► chunk ──► embed ──► prove ──► Polyp
  ├── ...
  └── form_40_integral.md       ──► chunk ──► embed ──► prove ──► Polyp
                                                                    │
                                                                    ▼
                                                          Reef (bedrock layer)
```

---

## 12. Crate Dependency Graph

Dependencies flow upward. No circular dependencies.

```
                         ┌──────────────┐
                         │  chitin-cli  │
                         └──────┬───────┘
                                │
                         ┌──────┴───────┐
                         │chitin-daemon │
                         └──┬────┬───┬──┘
                            │    │   │
              ┌─────────────┘    │   └─────────────┐
              │                  │                 │
      ┌───────┴──────┐  ┌────────┴───────┐  ┌──────┴──────────┐
      │  chitin-rpc  │  │chitin-consensus│  │chitin-economics │
      └───────┬──────┘  └───┬────────┬───┘  └───────┬─────────┘
              │             │        │              │
              │       ┌─────┘        └───┐          │
              │       │                  │          │
      ┌───────┴───────┴───┐    ┌─────────┴────┐     │
      │  chitin-reputation│    │ chitin-drift │     │
      └───────┬───────────┘    └──────┬───────┘     │
              │                       │             │
              │         ┌─────────────┘             │
              │         │                           │
      ┌───────┴─────────┴───────────────────────────┴──────┐
      │                                                    │
      │     ┌─────────────┐  ┌─────────────┐               │
      │     │ chitin-sync │  │chitin-verify│               │
      │     └──────┬──────┘  └──────┬──────┘               │
      │            │                │                      │
      │     ┌──────┴──────┐         │                      │
      │     │ chitin-p2p  │         │                      │
      │     └──────┬──────┘         │                      │
      │            │                │                      │
      │     ┌──────┴────────────────┴────────────────┐     │
      │     │           chitin-store                 │     │
      │     └──────────────────┬─────────────────────┘     │
      │                        │                           │
      └────────────────────────┼───────────────────────────┘
                               │
                        ┌──────┴──────┐
                        │ chitin-core │
                        └─────────────┘
```

### Dependency Matrix

| Crate | Depends On |
|---|---|
| `chitin-core` | (none — leaf crate) |
| `chitin-store` | `chitin-core` |
| `chitin-verify` | `chitin-core` |
| `chitin-p2p` | `chitin-core`, `chitin-store` |
| `chitin-sync` | `chitin-core`, `chitin-store`, `chitin-p2p` |
| `chitin-reputation` | `chitin-core`, `chitin-store` |
| `chitin-drift` | `chitin-core`, `chitin-store`, `chitin-verify` |
| `chitin-consensus` | `chitin-core`, `chitin-store`, `chitin-verify`, `chitin-reputation`, `chitin-drift` |
| `chitin-economics` | `chitin-core` |
| `chitin-rpc` | `chitin-core`, `chitin-store`, `chitin-consensus`, `chitin-economics`, `chitin-reputation` |
| `chitin-daemon` | all crates |
| `chitin-cli` | `chitin-core`, `chitin-rpc` (as RPC client) |

---

## 13. Design Decisions and Trade-offs

### Why Yuma Consensus (not PoS or BFT)?

**Decision**: Adapt Bittensor's Yuma Consensus rather than standard PoS/PBFT.

**Rationale**: Traditional PoS/BFT consensus (Tendermint, HotStuff) is designed for deterministic state machines — "did this transaction execute correctly?" Polyp validation is fundamentally different: it's a *quality judgment* with inherent subjectivity (is this text well-written? is this embedding useful?). Yuma Consensus is specifically designed for subjective quality evaluation in decentralized networks. Its stake-weighted median approach tolerates disagreement among validators while still converging on consensus. The bond mechanism incentivizes validators to develop expertise and stake their reputation on consistent, accurate scoring.

**Trade-off**: Yuma Consensus is more complex to implement and reason about than simple BFT. It introduces economic game theory that must be carefully tuned to avoid perverse incentives.

### Why ZK Proofs for Embeddings?

**Decision**: Require ZK proofs (SP1/Risc0) attesting that Vector = Model(Text).

**Rationale**: Without ZK proofs, a malicious Coral Node could submit arbitrary vectors paired with unrelated text — "semantic spam." Validators would need to re-run the embedding model for every Polyp to verify correctness, which is computationally expensive and defeats the purpose of distributed work. ZK proofs shift the cost to the prover (Coral Node) while making verification constant-time for validators.

**Trade-off**: ZK proving is currently expensive (seconds to minutes per proof for large models). This limits throughput in Phase 3. Mitigation: start with small models (bge-small-en, 384 dimensions) that prove quickly, and rely on hardware acceleration (GPU/FPGA provers) improving over time. Phase 1-2 can defer ZK proofs entirely.

### Why EMA Scoring?

**Decision**: Use Exponential Moving Average for validator-to-coral score histories.

**Rationale**: Per-epoch scores are noisy — a normally excellent Coral Node might have one bad epoch. EMA smooths this variance and rewards consistent quality over time. The alpha parameter (default 0.1) means approximately the last 10 epochs of performance are meaningfully weighted, preventing both stale scores and knee-jerk reactions.

**Trade-off**: EMA is slow to adapt to genuine quality changes. A Coral Node that suddenly starts producing bad Polyps will still ride on its historical reputation for several epochs. Mitigation: the ZK validity dimension (binary pass/fail) provides an immediate circuit-breaker — a single invalid proof triggers instant rejection regardless of EMA scores.

### Why Dual Storage (IPFS + Qdrant)?

**Decision**: Separate immutable storage (IPFS) from queryable index (Qdrant/HNSW).

**Rationale**: IPFS provides content-addressing and immutability (the "Chitin" guarantee) but is too slow for real-time ANN search (DHT lookups add seconds of latency). Qdrant provides millisecond-latency vector search but is mutable and not content-addressed. By separating the concerns, we get both guarantees: hardened Polyps are pinned to IPFS for permanent auditability, while the HNSW index provides fast retrieval for agents.

**Trade-off**: Two storage systems means more operational complexity, potential consistency issues (what if IPFS has the Polyp but the index doesn't?), and higher resource requirements per node. Mitigation: the hardening process atomically writes to both systems, and sync reconciliation (Section 4.2) detects and repairs inconsistencies.

### Why Rust + Python?

**Decision**: Core protocol in Rust, SDK in Python.

**Rationale**: Rust provides the performance, memory safety, and concurrency guarantees needed for the daemon (P2P networking, ZK verification, consensus computation). The AI agent ecosystem overwhelmingly uses Python (LangChain, LlamaIndex, CrewAI, AutoGen). A Python SDK lowers the barrier to adoption by meeting developers where they are.

**Trade-off**: Maintaining two language ecosystems increases development burden. The Python SDK must be kept in sync with Rust type changes. Mitigation: Protocol Buffers serve as the canonical type definitions — both Rust and Python types are generated from the same `.proto` files.

### Why 12 Crates (not fewer)?

**Decision**: Split the Rust workspace into 12 focused crates rather than a monolithic binary.

**Rationale**: Each crate maps to a distinct protocol concern (storage, networking, consensus, economics, etc.). This enables: (1) independent testing and auditing of each subsystem, (2) conditional compilation for different node roles (a Coral Node doesn't need `chitin-consensus` internals), (3) clear dependency boundaries that prevent architectural drift, (4) parallel compilation for faster build times.

**Trade-off**: More crates mean more `Cargo.toml` files, more cross-crate API surface to maintain, and longer initial project setup. For a team of 1-3 developers in Phase 1, this may be over-engineered.

---

## 14. Glossary

| Term | Definition |
|---|---|
| **ANN** | Approximate Nearest Neighbor — a search algorithm that finds vectors close to a query vector in sub-linear time. |
| **Attestation** | A validator's signed statement confirming a Polyp's quality and CID after hardening. |
| **Axon** | A node's inbound communication endpoint. Receives queries from other nodes. (Retained from Bittensor terminology.) |
| **Bond** | A validator's accumulated commitment to scoring a specific Coral Node. Built up via EMA over epochs. Higher bonds = higher dividend share. |
| **CID** | Content Identifier — IPFS's content-addressed hash used to uniquely identify immutable data. |
| **Chitin** | The hardened, finalized state of a Polyp. Also the name of the protocol and the token. |
| **Chitin Hardening** | The process of finalizing a Polyp: IPFS pin + on-chain CID anchor + Merkle proof + validator attestations. |
| **Coldkey** | Long-term identity key, kept offline. Controls staking and reward withdrawal. |
| **Consensus Weight** | The stake-weighted median score assigned to a Coral Node by the Yuma-Semantic Consensus algorithm. |
| **Coral Node** | A node that produces Polyps (text → embedding → ZK proof → submission). Analogous to a Bittensor Miner. |
| **$CTN** | The Chitin token. Used for staking, rewards, slashing, and governance. |
| **Dendrite** | A node's outbound communication endpoint. Sends queries to other nodes' Axons. (Retained from Bittensor terminology.) |
| **DID** | Decentralized Identifier — a self-sovereign identity standard (W3C). Format: `did:chitin:0x...` |
| **Dividend** | A Tide Node's share of epoch emission, proportional to agreement with consensus and bond strength. |
| **EMA** | Exponential Moving Average — `score = alpha * new + (1 - alpha) * old`. Used to smooth validator scores over time. |
| **Embedding** | A dense vector representation of text, produced by a neural network model. Maps semantic meaning to geometric proximity. |
| **Epoch** | A fixed-length period (default 360 blocks, ~1 hour) during which validation occurs and consensus is computed. |
| **HNSW** | Hierarchical Navigable Small World — a graph-based index structure for efficient ANN search. |
| **Hotkey** | Operational key, kept on the node. Signs messages and proofs. Compromise does not affect staked funds. |
| **Incentive** | A Coral Node's share of epoch emission, proportional to consensus weight (quality of produced Polyps). |
| **JSON-LD** | JSON for Linked Data — a method of encoding Linked Data using JSON. Used for Polyp schema interoperability. |
| **Kappa (κ)** | The consensus threshold in Yuma-Semantic Consensus. The stake-weighted median stops at this cumulative stake fraction (default 0.5). |
| **Metagraph** | See Reef Metagraph. |
| **Molting** | Re-embedding Polyps under a new model version. Named after arthropod exoskeleton shedding. |
| **Novelty** | A scoring dimension measuring how much new information a Polyp adds relative to existing hardened Polyps. |
| **Polyp** | The atomic unit of knowledge in Reefipedia: text + vector embedding + ZK proof + provenance metadata. |
| **PolypScores** | Five-dimensional quality assessment: ZK validity, semantic quality, novelty, source credibility, embedding quality. |
| **Proof of Semantic Integrity** | The protocol's novel consensus primitive: a ZK proof attesting that a vector was honestly generated from source text by a specific model. |
| **Provenance** | The full origin chain of a Polyp: who created it, from what source, through what processing pipeline. |
| **Rao** | The smallest unit of $CTN. 1 CTN = 10^9 rao. (Named after the unit in Bittensor.) |
| **Reef** | The aggregate of all hardened Polyps across the network — the global knowledge graph. |
| **Reef Metagraph** | The global network state: all nodes, stakes, trust scores, weights, bonds, Polyp counts. Updated every epoch. |
| **Reef Zone** | A topic-scoped partition of the knowledge graph (e.g., "medical", "code/python", "philosophy"). |
| **Semantic Drift** | The divergence in vector space geometry when embedding models are updated. Vectors from different models are not directly comparable. |
| **Semantic Sybil** | A malicious node that inserts plausible-but-misleading vectors into the index to manipulate retrieval results. |
| **Slashing** | Partial or full forfeiture of staked $CTN due to protocol violations (invalid proofs, consensus deviation, liveness failure, duplicates). |
| **Tide Node** | A node that evaluates Polyp quality via multi-dimensional scoring and participates in consensus. Analogous to a Bittensor Validator. |
| **TOFU** | Trust On First Use — Phase 1 trust model where all local submissions are accepted without verification. |
| **ValidationQuery** | A Tide Node's request to a Coral Node for Polyps to evaluate during an epoch. |
| **Venomx** | Vector Embedding Named Object Model indeX — a LinkML-based standard for embedding metadata. |
| **Weight** | A validator's score assignment to a Coral Node, submitted at the end of each epoch. Weights are inputs to Yuma-Semantic Consensus. |
| **Xi (ξ)** | The validator emission fraction — the share of epoch emission allocated to Tide Nodes (default 0.41). |
| **Yuma-Semantic Consensus** | The adapted Yuma Consensus algorithm used to reach agreement on Polyp quality. Uses stake-weighted median, weight clipping, bonds, and penalties. |
| **ZK Proof** | Zero-Knowledge Proof — a cryptographic proof that a computation was performed correctly without revealing the computation itself. Used here to prove Vector = Model(Text). |
| **zkVM** | Zero-Knowledge Virtual Machine (SP1, Risc0) — executes arbitrary code and generates a proof of correct execution. |
