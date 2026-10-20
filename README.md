# MBQ: Merge-Based and Majority-Quorum BFT Protocol

## Purpose
- This protocol aims to ensure safety in a Byzantine environment using only a majority quorum in a consensus algorithm. It addresses Byzantine consensus through the exchange of observations, unforgeable signatures, and the assumption of synchronous communication.

## Assumptions
- Byzantine nodes may crash, experience network failures or timeouts, remain partially or completely silent, equivocate, issue false requests, collude, and perform other such actions.
- The communication environment is assumed to be synchronous.
- This algorithm addresses only the consensus problem. Failure recovery is the responsibility of the implementation.
- There are at most `f` Byzantine, crash, or omission-fault nodes and `f+1` honest nodes, for a total of `2f+1` nodes.
- A valid client request may occasionally fail to complete, but it can succeed through retries.
- The retry policy is the client's responsibility.
- The client is assumed to always be honest and never Byzantine.
- All nodes and the client can produce unforgeable signatures.
- Load-balancing rules such as IP hashing, which always map a client to the same server in a 1:1 manner, are unsuitable for this approach.
- `Unknown` is independent of whether a node is Byzantine.
- Signatures are unforgeable.
- A given client request is committed at most once.
- Client retries are the implementation's responsibility. Idempotency must be preserved, for example, by checking the log for an existing instance of the same request.
- False failures may occur, and the policy for handling them is the client's responsibility. A false failure starts a new consensus instance unrelated to the previous consensus.
- Each signature covers all of its message's contents. A signature cannot be reused in another phase or for another request.
- A phase ends when responses have been received from all nodes or a timeout occurs. Messages arriving after the phase has ended are not accepted.
- Each consensus instance handles exactly one client request.
- The commit-certificate signatures in the Commit phase may be optimised using techniques such as threshold signatures, provided safety is preserved by the implementation.

## Phases

### Propose
```text
Propose(clientId, request, requestSignature) -> (proposeId)
```
- The node that initiates Propose becomes the proposer for this consensus instance.
- This phase forwards the request received from the client to the other nodes.
- The proposer advances to the next phase after receiving responses from all nodes, including timeout outcomes.
- Since one node propagates the request to the others, the time complexity is `O(n)`.

### Prepare
```text
Prepare(proposeId, sentRequest, isRequestSignatureValid, Signature)
    -> (sentRequest, isRequestSignatureValid, Signature)
```
- In this phase, each node exchanges the value it received from the proposer and whether the request's signature is valid.
- After this phase, each node has both the value it received and the values observed by other nodes.
- The node also provides a signature as evidence that the claim originated from that node, proving that the node produced the value.
- Each node must produce exactly one valid signature per consensus instance. Producing more than one valid signature is considered equivocation.
- A signature is always verified when received. If a node did not receive Propose but receives a Prepare containing a valid client signature, it joins the consensus. Its own claim is `Unknown`.
- Nodes that did not receive Propose participate in subsequent phases if they receive a Prepare containing a valid client signature.
- A Prepare response concerning an unknown proposal always returns `Unknown`.
- The time complexity is `O(n^2)`.

### Merge
```text
Merge(proposeId,
      {clientId: [request, isRequestSignatureValid, prepareSignature]}, ...)
    -> ({nodeId: {clientId: [request, isRequestSignatureValid, prepareSignature]}, ...})
```
- In this phase, nodes exchange the values they have observed.
- Each node includes its own observation. The observations of all nodes and the signatures serving as evidence for them are exchanged.
- A single Prepare phase does not guarantee safety because a Byzantine node may equivocate. This is addressed by comparing the observations with one another.
- This can be viewed as increasing the dimensionality of the values held by the nodes.
- A Byzantine node may equivocate during this phase and cause different nodes to receive different information. However, this does not affect the final safety property.
- If a Merge value is not received because of silence, a timeout, or similar circumstances, it is treated as `Unknown`.
- **[9]** A claim made by an honest node during Prepare is shared identically with all honest nodes through Merge. Equivocation during Prepare is prevented from compromising the protocol by equivocation detection in the Merge and Conflict phases.
- The time complexity is `O(n^2)`. To reduce bit complexity, a fast path using hash verification could be implemented. Since this would not change the logic, it is left to the implementation.

### Conflict
- Broadly speaking, this phase is part of Merge, but it is separated for convenience.
- It begins once all value exchanges in Merge have finished.
- Based on the Merge results, each node determines the final claim for every node. This is performed locally.
- A conflicting claim may have a signature. If the claim is malicious and was produced by a Byzantine node, that signature may be false.
- If two or more valid signatures for the same node refer to different values, the only possible explanation is that the node equivocated. The node is therefore treated as Byzantine and ignored, and its final claim is set to `Unknown`.
- Since honest nodes share signature evidence during Merge, if a Byzantine node signs and sends different values to honest nodes, all honest nodes can learn that it equivocated.
- For a node that passes equivocation detection, a claim containing a valid signature is accepted as that node's final claim. If no valid signature exists, its final claim is assumed to be `Unknown`.
- Even if Byzantine nodes equivocate during Merge, at most `f` Byzantine nodes can interfere. They therefore cannot corrupt honest nodes' claims or cause different honest nodes to adopt conflicting claims.
- Once the final claims of all nodes have been determined, a candidate value supported by a strict majority (`f+1`) is selected as the final Conflict result. If no candidate obtains a strict majority, the result is `Reject`.

### Commit
```text
Commit(proposeId, clientRequest, clientSignature, nodeConsensusSignature)
    -> (nodeConsensusSignature)

commitCert = {
    proposeId,
    clientRequest,
    clientSignature,
    {
        nodeId: nodeSignature,
        ...
    }
}
```
- The final commit decision is made based on the result of the Conflict phase.
- If a node's claim is treated as `Unknown` because of signature-verification failure, a timeout, silence, equivocation, or similar circumstances, it is considered not to have voted for any candidate value.
- Even if `Unknown` occurs, the overall quorum threshold (`Q=f+1`) and the value of `f` remain fixed; neither is reduced.
- If Conflict determines a valid candidate value (anything other than `Reject`), honest nodes sign that value and propagate their signatures to one another. If the result is `Reject`, they do not sign.
- All honest nodes derive the same valid candidate value or derive `Reject`. Since there are at most `f` Byzantine nodes, they cannot reach the strict-majority threshold of `f+1` on their own and therefore cannot commit conflicting values.
- Each node signs the information required to identify the client request, including `proposeId`, `clientRequest`, and `clientSignature`, using its private key, and provides the signature. A receiving node verifies the signature.
- If at least `f+1` valid commit signatures from distinct nodes are collected for a value, the result is accepted as a valid commit (a commit certificate).
- A consensus instance has only one candidate value. Since the `f` Byzantine nodes alone cannot reach the quorum of `f+1`, safety is preserved.
- Honest nodes do not sign different values, and Byzantine nodes alone cannot provide `f+1` signatures. Therefore, two conflicting commits cannot coexist.
- The client can determine whether a value was properly committed by receiving `f+1` commit signatures (a commit certificate).
- Since all nodes provide their signatures to one another, the communication complexity is `O(n^2)`.

# MBQ Lemmas

## State Definitions and Common Notation

The following states have distinct meanings.

- **`Unknown`**: A node has not obtained a required response or valid evidence (for example, because Propose was not received, a timeout occurred, a node remained silent, or signature verification failed), or equivocation was detected through conflicting signatures, so the node cannot support a particular candidate. `Unknown` is not counted as support for any candidate and does not, by itself, establish that the node is faulty. The overall quorum threshold (`Q=f+1`) is not reduced when `Unknown` occurs.
- **`Reject`**: The local decision produced in the Conflict phase when no candidate obtains a strict majority (`Q=f+1`). An honest node that derives `Reject` does not generate a Commit signature.

Common notation:

- `n=2f+1`: The size of the complete node set `V` (`|V|=2f+1`).
- `f`: The upper bound on the number of faulty nodes (Byzantine, crash, or omission faults).
- `Q=f+1`: The strict-majority quorum size.
- `H`: The set of honest nodes (`|H| >= f+1`).
- `F`: The set of faulty nodes (`|F| <= f`, `H ∩ F = ∅`, and `H ∪ F = V`).
- `𝒱`: The set of candidate request values with valid client signatures.
- `P_k ∈ 𝒱 ∪ {Unknown}`: The claim that honest node `k ∈ H` creates for itself during Prepare. If it receives Propose correctly, this is the request value `v ∈ 𝒱`. If it does not receive Propose but joins through a Prepare containing a valid client signature, its claim is `Unknown`.
- `E_k(v)=(v, σ_k(v))`: The pair consisting of node `k ∈ V`'s Prepare claim for candidate value `v ∈ 𝒱` in the current consensus instance (`proposeId`) and the node's valid signature. The signature `σ_k(v)` includes the consensus-instance and phase information and cannot be reused in another phase or for another request.
- `M_i`: The set of signed evidence and observation reports collected by honest node `i ∈ H` by the deadlines of the Prepare and Merge phases.
- `c_i(k) ∈ 𝒱 ∪ {Unknown}`: The final Prepare claim for node `k ∈ V` that honest node `i ∈ H` determines during Conflict based on `M_i`.
- `C_i ∈ 𝒱 ∪ {Reject}`: The final Conflict result derived by honest node `i ∈ H`.

Each node is counted as a supporter of at most one candidate in a given consensus instance. Conflicting claims from the same node are not counted as independent support for different candidates.

## Lemma A — Detection of Equivocation Through Conflicting Signatures

**Claim.** If a node produces valid signatures for two conflicting claims in the same consensus instance, and both signatures are provided to a verifier, the node's equivocation can be detected deterministically.

**Proof.**

Suppose node `b ∈ V` produces valid signatures `σ_b(v₁)` and `σ_b(v₂)` for two different values `v₁, v₂ ∈ 𝒱` in the same consensus instance, where `v₁ ≠ v₂`.

Since signatures cannot be forged, and both signatures verify under node `b`'s public key, the existence of the two valid signatures for different values proves that node `b ∈ F` equivocated, because an honest node creates signatures for only one value in the same consensus instance. A verifier that obtains both signatures can compare them, immediately detect the equivocation, and invalidate the node's claim by setting it to `Unknown`.

**Conclusion.** Once both conflicting valid signatures have been obtained, equivocation can be detected deterministically.

## Lemma B — Evidence Sharing Among Honest Nodes After Merge and Compatibility of Per-Node Claims

**Claim.** Suppose all honest nodes broadcast during Merge the sets of valid signature evidence they created or observed during Prepare, and messages between honest nodes are delivered by the specified deadline. Each honest node `i ∈ H` determines the final claim `c_i(k)` for any node `k ∈ V` from its collected Merge set `M_i` according to the following rules:

1. **Signature-verification rule:** Exclude any false report about node `k` in `M_i` for which neither a valid client signature nor a valid Prepare signature `σ_k(v)` from node `k` can be verified.
2. **Equivocation-invalidation rule:** If `M_i` contains two or more valid signature-evidence items `E_k(v₁)` and `E_k(v₂)` for the same node `k` that refer to different values (`v₁ ≠ v₂`), treat node `k` as a Byzantine node that equivocated and set `c_i(k)=Unknown`.
3. **Final-claim rule:** If there are no conflicting valid signatures (that is, equivocation detection is passed), set `c_i(k)=v` only if at least `Q=f+1` distinct nodes report the same valid signature evidence `E_k(v)` during Merge. Otherwise, set `c_i(k)=Unknown`.

Under these conditions, even if a Byzantine node equivocates during Prepare or Merge, it cannot corrupt honest nodes' claims. It is impossible for two different honest nodes `i,j ∈ H` to simultaneously finalise conflicting candidate values `v₁ ≠ v₂` for the same node `k`.

**Proof.**

First, signatures cannot be forged. Therefore, even if a Byzantine node sends a fabricated claim about node `k` during Merge, it cannot provide a valid signature `σ_k(·)`, and the claim is excluded by the **signature-verification rule**. In particular, an honest node `h ∈ H` makes at most one claim `P_h ∈ 𝒱 ∪ {Unknown}` in the same consensus instance and does not create conflicting signatures. Thus, Byzantine equivocation during Merge cannot cause an honest node's claim to be falsely classified as equivocation or be altered to another value.

Suppose an honest node `p ∈ H` observed node `k`'s valid signature evidence `E_k(v)=(v, σ_k(v))` during Prepare. Node `p` broadcasts this evidence during Merge. Under the synchronous-network assumption, `p`'s Merge message reaches every honest node before the deadline. Therefore,

`∀ p,i ∈ H, E_k(v) ∈ M_p ⇒ E_k(v) ∈ M_i`.

For a proof by contradiction, suppose two honest nodes `i,j ∈ H` finalise different valid candidate values `v₁,v₂ ∈ 𝒱` for the same node `k`, where `v₁ ≠ v₂`:

`c_i(k)=v₁` and `c_j(k)=v₂`.

By the **final-claim rule**, node `i` must have received reports of `E_k(v₁)` from at least `f+1` nodes, and node `j` must have received reports of `E_k(v₂)` from at least `f+1` nodes.

Since there are at most `f` faulty nodes (`|F| ≤ f`), even if Byzantine nodes equivocate during Merge, they can manipulate at most `f` reports. Thus, among the at least `f+1` nodes reporting `E_k(v₁)`, there must be at least one honest node `p ∈ H` that observed it during Prepare. Similarly, among the at least `f+1` nodes reporting `E_k(v₂)`, there must be at least one honest node `q ∈ H` that observed it during Prepare.

The evidence `E_k(v₁)` and `E_k(v₂)` broadcast by honest nodes `p` and `q` during Merge reaches every honest node by the deadline. Therefore, both evidence items must be included in the evidence sets of nodes `i` and `j`:

`{E_k(v₁), E_k(v₂)} ⊆ M_i ∩ M_j`.

By Lemma A and the **equivocation-invalidation rule**, nodes `i` and `j` should therefore both have finalised `c_i(k)=Unknown` and `c_j(k)=Unknown`, contradicting the assumption.

**Conclusion.** For any node `k ∈ V`, there is at most one candidate value `v*_k ∈ 𝒱` such that the final claim for node `k` determined by every honest node satisfies:

`∀ i ∈ H, c_i(k) ∈ {v*_k, Unknown}`.

In particular, an honest node `k ∈ H` does not equivocate and communicates its Prepare state `P_k ∈ 𝒱 ∪ {Unknown}` to all honest nodes (`|H| ≥ f+1`). Therefore, every honest node `i ∈ H` determines `c_i(k)=P_k`.

## Lemma C — Quorum Limit of Faulty Nodes

**Claim.** If there are at most `f` faulty nodes and both a majority quorum and a Commit certificate require support or signatures from `Q=f+1` distinct nodes, faulty nodes alone cannot satisfy the quorum or construct a Commit certificate.

**Proof.**

The number of faulty nodes and the quorum size satisfy:

`|F| ≤ f < f+1 = Q`.

Therefore, faulty nodes alone cannot provide support or signatures from `f+1` distinct nodes.

`Unknown` is not counted as support for any candidate. Conflicting claims from the same node are also not counted as independent support for different candidates. Equivocation is detected as described in Lemma A.

`Unknown` is not, by itself, evidence that a node is faulty. Neither `f` nor the quorum threshold `Q=f+1` is reduced because `Unknown` or equivocation occurs.

**Conclusion.** Claims or signatures from faulty nodes alone cannot satisfy the majority quorum in Conflict or the certificate threshold in Commit.

## Lemma D — Safety of the Commit Certificate

**Claim.** If every honest node `i ∈ H` generates a Commit signature for its Conflict result only when `C_i` is a valid candidate value (`C_i ≠ Reject`), two valid Commit certificates (`commitCert`) for different values `v₁ ≠ v₂` cannot be generated in the same consensus instance.

**Proof.**

By Lemma B, honest nodes can derive at most one valid candidate value other than `Reject` in a single consensus instance, denoted by `v* ∈ 𝒱`. An honest node generates and propagates a Commit signature (`nodeConsensusSignature`) using its private key, binding together `proposeId`, `clientRequest`, and `clientSignature`, only for its own valid result `C_i=v*`. It does not sign if `C_i=Reject`. Therefore, no honest node generates a Commit signature for a conflicting value `v' ≠ v*`, where `v' ∈ 𝒱`.

A valid Commit certificate (`commitCert`) for the conflicting value `v'` requires at least `Q=f+1` valid Commit signatures from distinct nodes. Since honest nodes do not sign `v'`, only faulty nodes could sign it.

Therefore, by Lemma C, the number of available signatures satisfies:

`#Signers(v') ≤ |F| ≤ f < f+1 = Q`.

This contradicts the requirement for at least `f+1` signatures to form a Commit certificate.

**Conclusion.** Two valid Commit certificates for different values cannot be generated in the same consensus instance.

A Commit certificate (`commitCert`) consists of `proposeId`, `clientRequest`, `clientSignature`, and a set of `nodeConsensusSignature` values from at least `f+1` distinct nodes. The client receives and verifies the certificate to determine whether the request was properly committed.

## Lemma E — Safety Against a Byzantine Proposer

**Claim.** Even if a Byzantine proposer sends a request to only some nodes, sends conflicting requests to different nodes, or stops forwarding the request, valid Commit certificates for different values cannot be generated under the conditions of the preceding lemmas.

**Proof.**

The client is always honest and signatures cannot be forged. Therefore, the proposer cannot forge the client's signature to arbitrarily modify the contents of a signed request.

If the proposer sends the request to only some honest nodes or remains silent, an honest node that did not directly receive Propose can set its own claim to `Unknown` and participate normally in the subsequent consensus phases (Prepare, Merge, Conflict, and Commit) after receiving a Prepare message containing a valid client signature. Furthermore, even if the proposer sends conflicting requests to different nodes or sends a request to only some of them, the Prepare results created and observed by honest nodes are shared with all honest nodes during Merge under the conditions of Lemma B.

Therefore, by Lemma D, the Conflict results of all honest nodes are restricted to `C_i ∈ {v*, Reject}` for a single candidate value `v*`. Consequently, no Commit certificate can be generated for another value `v' ≠ v*`. (Even if a false failure occurs because some honest nodes derive `C_i=v*` while others derive `Reject`, so that fewer than `f+1` Commit signatures are collected, safety is not violated: two conflicting values are never committed.)

**Conclusion.** Under the conditions of the proof, selective forwarding, equivocation, or silence by a Byzantine proposer cannot break Commit safety.

This lemma concerns safety. Progress after failures, including false failures, and eventual success through client retries are addressed in Lemma G.

## Lemma F — Validity and Probabilistic Progress Through Honest-Proposer Selection

**Claim.** In a consensus instance with an honest proposer, a valid Commit certificate is always generated for the client's valid request value `v` (validity). In addition, if the client avoids fixed 1:1 mappings such as IP hashing and, for each retry, independently and uniformly selects a proposer from all `n=2f+1` nodes for a new consensus instance, then, with at most `f` faulty nodes, the probability that an honest proposer is selected and the request succeeds on each attempt satisfies:

`p ≥ (f+1)/(2f+1) > 1/2`.

**Proof.**

If the selected proposer is honest (`∈ H`), it sends the client's valid request `v` to all honest nodes before the Propose deadline. Every honest node `k ∈ H` creates and shares the same Prepare evidence `E_k(v)` (`P_k=v`). By Lemma B, every honest node `i ∈ H` determines `c_i(k)=v` for every honest node `k ∈ H`.

There are at least `|H| ≥ f+1 = Q` honest nodes. Thus, for every honest node `i ∈ H`, the support set for `v` satisfies `|S_i(v)| ≥ f+1`, and `C_i=v` is derived. Therefore, at least `f+1` honest nodes generate Commit signatures for `v`, so a valid Commit certificate (`commitCert`) is deterministically generated within a single round.

Meanwhile, there are `2f+1` nodes in total and at most `f` faulty nodes. Under uniform random selection without fixed 1:1 load balancing, the probability of selecting an honest proposer is:

`p = |H|/(2f+1) ≥ (f+1)/(2f+1) > 1/2`.

If each retry is performed independently in a new consensus instance (`proposeId`) unrelated to the previous one, the probability that an honest proposer is selected at least once within `k` attempts satisfies:

`P(honest proposer selected) ≥ 1 - (f/(2f+1))^k`.

Therefore, as `k` increases, the probability of selecting an honest proposer at least once converges to `1`.

**Conclusion.** When an honest proposer is selected, consensus is reached deterministically within a single round. Under a random proposer-selection policy, each attempt has a success probability greater than one half, and independent retries can raise the probability of success arbitrarily close to one.
