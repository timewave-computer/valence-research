# Recursive ZK Light Client for Ethereum

## Context

At **Timewave**, we are dedicated to addressing complex challenges in **interchain development**, with the goal of powering a new generation of **trustless applications**. These applications are designed to seamlessly access **liquidity across multiple blockchain ecosystems** in parallel.

Our modular architecture, embodied in the **Valence** development stack, enables the integration of heterogeneous networks—regardless of their consensus mechanism or architecture—provided they support a minimal set of **cryptographic operations**.

These operations are essential for describing and proving the correctness of a network’s **consensus protocol**. Whether a network uses **Proof-of-Work (PoW)**, **Proof-of-Stake (PoS)**, or another mechanism is secondary, as long as the protocol’s rules can be externally verified via **cryptographic proof**. This forms the foundation for building a **Zero-Knowledge (ZK) Light Client**.

In addition to a well-defined consensus protocol, the network must expose **state proofs**. A ZK Light Client provides a **mathematically verifiable block** without requiring trust. A block is defined by its `number` and its `root`. The `root` reflects the internal state of the blockchain and enables verification of any data or value stored at that point in time. The **state proof** is the cryptographic artifact used to verify data against that root.

To verify state correctness across any network, a ZK Light Client must:

- Provide a provably correct block (`root`, `number`)
- Accept requests for **state proofs** (e.g., balances or smart contract states), which can be verified against the `root` of that block

**→ Key Concepts:**  
- **ZK Light Client**: Proves blockchain state and consensus correctness with no trust assumptions  
- **State Proof**: Cryptographic proof of any specific state data (e.g., balance, contract storage)  
- **Root**: Merkle root representing the entire blockchain state at a given block  

---

## SP1-Helios Checkpoints

[**SP1 Helios**](https://github.com/succinctlabs/sp1-helios) is a **ZK Light Client for Ethereum**, developed by [Succinct](https://succinct.xyz), built on top of [A16z Helios](https://github.com/a16z/helios)—a traditional non-ZK Ethereum light client. SP1 wraps A16z Helios in a zero-knowledge proof system using the **SP1 ZKVM**, enabling **trustless verification** of Ethereum blocks.

SP1-Helios progresses through **Light Client Updates**, transitioning from a **trusted checkpoint** to a later **finalized block**. Only the first checkpoint requires trust; all future transitions are cryptographically verified against Ethereum's consensus rules.

```text
Trusted Checkpoint → Valid Target → New Trusted Checkpoint → Valid Target → ...
```

Each new block verified becomes the new checkpoint, maintaining a recursive chain of trust.

**→ Key Concepts:**  
- **Checkpoint**: A known-valid block serving as a verification anchor  
- **Target**: A later finalized block that has been verified  
- **ZKVM**: Zero-Knowledge Virtual Machine, executes verifiable logic  

---

## Ethereum Finality and Committees

Ethereum’s consensus is structured around **epochs** and **slots**:

- **1 slot** = `12 seconds`  
- **1 epoch** = `32 slots` ≈ `6.4 minutes`  
- **Committee rotation** = every `256 epochs` ≈ `27.3 hours`

A **sync committee**—a subset of validators—signs finality updates **once per epoch**. This committee rotates periodically, ensuring decentralized and dynamic participation.

When SP1-Helios generates a proof, it records the **previous committee** responsible for signing the current trusted block:

```rust
let prev_sync_committee_hash = store.current_sync_committee.tree_hash_root();
```

It then applies updates and transitions to the new block:

```rust
for (index, update) in updates.iter().enumerate() {
    println!("Verifying update {} of {}.", index + 1, updates.len());
    verify_update(update, expected_current_slot, &store, genesis_root, &forks)
        .expect("Update is invalid!");
    apply_update(&mut store, update);
}
```

Finally, it records the **new committee** that signed the latest block:

```rust
let sync_committee_hash: B256 = store.current_sync_committee.tree_hash_root();
```

**→ Key Concepts:**  
- **Slot**: 12-second window for proposing and attesting a block  
- **Epoch**: A group of 32 slots  
- **Sync Committee**: The validator subset responsible for finality signatures  

---

## Recursion

One of the key differences between **SP1-Helios** and our implementation—**Valence-Helios**—is how **future sync committees** are handled.

SP1-Helios records the **next committee** once it becomes known:

```rust
let next_sync_committee_hash: B256 = match &mut store.next_sync_committee {
    Some(next_sync_committee) => next_sync_committee.tree_hash_root(),
    None => B256::ZERO,
};
```

It also deploys an [on-chain verifier contract](https://github.com/succinctlabs/sp1-helios/blob/main/contracts/src/SP1Helios.sol) to maintain a mapping of all known committees. This allows **future proofs** to be accepted by the smart contract, even **before** the rotation takes effect.

### Stateless Recursive Verification

In contrast, **Valence-Helios** uses a **recursive, off-chain prover** that maintains only the **most recent proof** in memory. This proof becomes the **trusted checkpoint** for the next step. No on-chain verification is required.

This design allows us to:

- Eliminate the need to store sync committees on-chain
- Minimize gas costs (no on-chain updates)
- Generate and serve updated proofs **on demand**

**→ Key Concepts:**  
- **Recursive Proofs**: Each proof builds upon a previously verified state  
- **Stateless Verification**: No persistent smart contract storage needed  
- **On-demand Proving**: The prover updates in real-time, without transactions  

---

## Integration

Integrating Helios into the Valence stack was straightforward. The [**Valence Coprocessor**](https://github.com/timewave-computer/valence-coprocessor) provides an API to register new domains—like `Ethereum`.

Once registered, we connected our recursive Helios prover to the Coprocessor by sending finalized block ZK proofs to its endpoint using a lightweight [**relayer service**](https://github.com/timewave-computer/helios-proof-relayer/blob/392686582b73685fa28ebe9139b50ac6abf2b5ab/src/main.rs#L12). This relayer is optional; direct communication is also possible.

The [**Valence-Helios**](https://github.com/timewave-computer/valence-helios) operator exposes a simple HTTP API:

```
GET /
```

This endpoint returns the latest **finalized block's zero-knowledge proof**, enabling applications to **trustlessly verify Ethereum state** in real-time.

**→ Key Concepts:**  
- **Coprocessor**: All in one ZK proving service for interchain app development 
- **Relayer**: A simple service to forward ZK proofs from the prover to the API  
- **API Endpoint**: `get_proof` returns the latest Ethereum ZK proof  


## Call to Action: Integrate New Domains

The Valence architecture is designed to be modular, extensible, and ZK-native.

If you're building or operating a blockchain that exposes verifiable state and has a transparent consensus protocol, you can integrate a **ZK Light Client** for your chain into Valence with minimal friction.

To add a new domain:

1. Implement or connect a light client (ZK or traditional) for your network.
2. Register the domain with the [Valence Coprocessor](https://github.com/timewave-computer/valence-coprocessor).
3. Submit finalized block proofs and state proofs via the Coprocessor API.

**→ Want to integrate your chain?**  
Check out the [Valence Coprocessor repo](https://github.com/timewave-computer/valence-coprocessor) or reach out directly — we’re here to help.

**Let's expand the interchain ZK ecosystem together.**
