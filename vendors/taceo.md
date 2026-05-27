---
title: "Vendor: TACEO"
status: draft
maturity: pilot
---

# TACEO

TACEO is a privacy infrastructure provider building MPC, threshold
cryptography, and collaborative proving for EVM chains. Two solutions sit on
its distributed MPC network: **Private Finance** (TACEO Merces), a
confidential token protocol with a UC security proof, and **Private Identity**
(the TACEO Identity stack), production-grade privacy-preserving identity
infrastructure. The same network underpins both: independent operators
jointly execute on secret-shared inputs and emit onchain CoSNARKs verifying
state transitions, with no single operator seeing plaintext.

## Private Finance (TACEO Merces)

### What it is

TACEO Merces is an implementation of **Private Shared State (PSS)** for
confidential token transfers on EVM chains. It combines multiparty
computation (MPC) with zero-knowledge proofs (Groth16 via co-SNARKs) to hide
transfer amounts (confidential mode) or both transfer amounts and party
addresses (fully private mode, via the TACEO:OMap data structure) while
maintaining onchain verification. The protocol is published (IACR ePrint
2026/850) with a full UC security proof. Deployed on Base (including Base
Sepolia for the x402 integration), Arc, and Plasma testnets; mainnet
deployment in progress.

### Fits with patterns

- [co-SNARKs (Collaborative Proving)](../patterns/pattern-co-snark.md): Core proving mechanism; MPC nodes jointly generate Groth16 proofs
- [Private Shared State (co-SNARKs)](../patterns/pattern-private-shared-state-cosnark.md): Merces is the canonical implementation
- [Shielding](../patterns/pattern-shielding.md): Balances stored as Pedersen commitments; amounts hidden from observers (confidential mode)
- [Private Stablecoin Shielded Payments](../patterns/pattern-private-stablecoin-shielded-payments.md): Confidential transfer amounts (partial fit; see limitations below)
- [Regulatory Disclosure (viewing keys + ZK proofs)](../patterns/pattern-regulatory-disclosure-keys-proofs.md): Scoped audit access for regulators, downstream of default privacy, covered by the UC security proof
- [Verifiable Attestation](../patterns/pattern-verifiable-attestation.md): Per-transaction policy attestations (sanctions, KYC) verified onchain via Predicate integration
- [Noir Private Contracts](../patterns/pattern-noir-private-contracts.md): Noir circuits compiled via CoNoir to Groth16 proofs

### Not a substitute for

- **External anonymity infrastructure (mixers, anonymity sets):** Merces handles privacy at the protocol layer via secret-shared state; users who want privacy at the wallet or routing layer should combine Merces with separate infrastructure
- **Stealth addresses (ERC-5564):** Wallet-level recipient unlinkability requires separate stealth address infrastructure; Merces hides addresses at the protocol layer in fully private mode but does not replace wallet-derivation patterns

### Architecture

Three-component system:

1. **Smart Contract:** Holds ERC-20 tokens, maintains balance commitments, queues actions, verifies Groth16 proofs
2. **MPC Network:** Three computing nodes plus orchestration server; maintains secret-shared balance maps; generates co-SNARK proofs
3. **ZK Circuit:** Noir circuits compiled to Groth16; proves balance validity, non-negativity, and correct state transitions

**Transaction flow:**

1. User initiates transfer with recipient and amount
2. Wallet encrypts amount using Diffie-Hellman key exchange (BabyJubJub curve)
3. Orchestration server distributes encrypted shares to MPC nodes
4. Nodes collaboratively compute new balances and generate Groth16 proof
5. Proof and new balance commitments posted onchain

### Privacy domains

Merces supports two transfer modes, selectable per transaction.

**Confidential mode:**

- *Hidden:* transfer amounts (encrypted via DH, proven in ZK); individual account balances (Pedersen commitments)
- *Public:* sender and receiver addresses; deposit and withdrawal amounts; transaction existence and ordering

**Fully private mode (via TACEO:OMap):**

- *Hidden:* sender address, receiver address, transfer amount, account balances
- *Public:* state commitment updates (Merkle roots); deposit and withdrawal amounts at the pool boundary; transaction existence and ordering

**Disclosable on demand (both modes):**

- Scoped read access for regulators or auditors via the Regulatory Disclosure capability (downstream of default privacy, covered by the UC security proof)

### Enterprise demand and use cases

- **Confidential treasury operations:** Corporate payments where amounts are business-sensitive but counterparties are known
- **Interbank settlement:** Banks settling known bilateral positions without revealing volumes to third parties
- **Payroll and vendor payments:** Hide payment amounts from blockchain observers while maintaining compliance records
- **Machine-mediated commerce:** Confidential settlement on the x402 protocol for agent-to-agent payments and dynamic API pricing without exposing pricing strategies onchain
- **Yield on confidential balances:** Earn flows on shielded positions (Private Earn, in development)

### Technical details

| Metric | Value |
|--------|-------|
| Proof system | Groth16 (via CoNoir compiler) |
| Commitment scheme | Pedersen commitments |
| Encryption | BabyJubJub elliptic curve (DH key exchange) |
| Batch size | 50 transactions per proof |
| Constraints | ~79,400 per batch |
| Gas cost | ~95k per tx; ~3.8M per batch verification; single-digit cents per transfer on L2 |
| Throughput | ~300 TPS |
| MPC topology | 3-party replicated secret sharing (honest-majority, semi-honest) |
| Networks | Base, Arc, Plasma testnets; mainnet in progress |
| Demo throughput | ~5M transactions processed across Arc and Base testnets |

### Strengths

- **Two-mode privacy:** Confidential and fully private modes selectable per transaction; the latter hides sender, receiver, and amount at the protocol layer
- **ERC-20 compatible:** Works with existing stablecoin contracts
- **Batched proving:** reduces per-transaction costs
- **co-SNARK approach:** distributes trust across MPC nodes
- **Noir-based circuits:** Growing ecosystem and tooling
- **Scoped regulatory disclosure:** Viewing key infrastructure for auditors and entitled parties; the disclosure path is covered by the UC security proof
- **Composes with external policy providers:** Per-transaction policy attestations (sanctions, KYC) verified onchain via partnership with Predicate

### Risks and open questions

- **Honest-majority MPC assumption:** Protocol uses 3-party replicated secret sharing in the semi-honest setting; confidentiality holds as long as at least two of the three parties remain honest. Two colluding parties can reconstruct secrets.
- **Batch latency:** Transactions wait for batch fill (50 txs) before finalization. If a single transaction in a batch fails, the whole transaction batch fails, affecting liveness considerations.
- **Testnet only:** Mainnet deployment in progress; production security at scale unproven
- **Deposit/withdrawal leakage:** Entry/exit amounts public; flow analysis possible at the pool boundary
- **Orchestration trust:** Demo setup holds plaintext balances; production requires additional encryption

## Private Identity (TACEO Identity)

### What it is

TACEO Identity is a privacy-preserving identity stack running on the TACEO
Network. It combines threshold vOPRF for issuer-independent nullifier
generation, MPC biometric matching for uniqueness checks without template
exposure, and collaborative proving via co-SNARKs. The stack is in production
use, powering uniqueness verification and document-based identity checks for
large-scale identity systems.

### Fits with patterns

- [vOPRF Nullifiers](../patterns/pattern-voprf-nullifiers.md): Threshold vOPRF generates deterministic per-scope unlinkable nullifiers without any operator learning the user's identifier
- [co-SNARKs (Collaborative Proving)](../patterns/pattern-co-snark.md): Same MPC and collaborative proving substrate as Merces

### Architecture

Three composable services run on the TACEO Network. **TACEO:OPRF** operates
a threshold vOPRF that generates deterministic per-scope unlinkable
nullifiers from a user's base identifier, without any operator learning the
identifier (public endpoint at eu.oprf.taceo.network). **TACEO:Match**
performs MPC-based comparison of biometric templates without any party
reconstructing either template in plaintext. **TACEO:Proof** binds the
outputs into onchain verifiable claims via collaborative SNARKs.

Each service is independently composable. Biometric uniqueness combines OPRF
and Match. Document-based credential systems compose OPRF with their own
credential proofs for unlinkable per-scope nullifiers.

### Strengths

- **Production at scale:** Live on large-scale identity deployments in production
- **Plural enrollment:** Passport, biometric, and other credential sources compose independently
- **Issuer-independent:** Post-enrollment verification needs no contact with the original credential issuer

### Risks and open questions

- Threshold OPRF network requires committee governance and share rotation
- Selective disclosure capability is roadmap, not shipped
- No standalone Identity protocol paper

## CROPS profile

| Solution          | CR     | OS      | Privacy | Security | Context |
|------------------|--------|---------|---------|----------|---------|
| Private Finance (TACEO Merces)  | medium | partial | full    | medium   | i2i     |
| Private Identity | medium | partial | full    | medium   | both    |

Both solutions share the same trust profile because they share the same
network. Privacy is `full` (disclosure modes are available but not required
for participation); Security is `medium` (rides on honest-majority MPC); CR
is `medium` (participation requires the operator set, no single party can
exclude a user); OS is `partial` (the Merces paper and the co-snarks
framework are open under permissive licences; production orchestration
includes proprietary components).

## Links

- [TACEO](https://taceo.io)
- [Merces paper (IACR ePrint 2026/850)](https://eprint.iacr.org/2026/850)
- [Merces documentation](https://merces-demo.taceo.io/base/introduction/overview)
- [Confidential x402 repo](https://github.com/TaceoLabs/merces1-x402)
- [TACEO:OMap article](https://core.taceo.io/articles/taceo-omap/)
- [CoSNARKs repo](https://github.com/TaceoLabs/co-snarks)
- [TACEO coSNARK reference](https://core.taceo.io/)
- [TACEO:OPRF endpoint](https://eu.oprf.taceo.network)
- [OPRF technical writeup](https://core.taceo.io/articles/taceo-oprf/)
- [Private proof delegation writeup](https://core.taceo.io/articles/private-proof-delegation/)