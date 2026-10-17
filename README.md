# MBQ: Merge-Based and Majority-Quorum BFT Protocol

## Purpose

- This protocol aims to achieve safety in a Byzantine environment using only a majority quorum. It addresses Byzantine consensus through the exchange of observations, unforgeable signatures, and the assumption of synchronous communication.

## Assumptions

- Byzantine nodes may crash, remain partially or completely silent, equivocate, submit false requests, collude, and perform other Byzantine behaviors.
- The communication environment is assumed to be synchronous.
- There are \(f\) nodes that may be Byzantine, crash, or omit messages, and \(f+1\) honest nodes, for a total of \(2f+1\) nodes.
- A valid client request may fail to complete in some circumstances, but it may succeed through retries.
- The client is responsible for the retry policy.
- The client is always honest and is assumed never to be Byzantine.
- All nodes and clients can generate unforgeable digital signatures.
- Load-balancing policies such as IP hashing, which consistently map a client to a single server in a one-to-one manner, are unsuitable for this approach.
- `Unknown` does not imply that a node is Byzantine.
- Signatures are unforgeable.
- False failures may occur, and the client is responsible for the policy governing their handling.
## Phases

### Propose

```text
Propose(clientId, request, requestSignature) -> (proposeId)
```

- The node that initiates the proposal becomes the proposer for the current consensus instance.
- This phase forwards the request received from the client to the other nodes.
- The proposer proceeds to the next phase after receiving responses from all nodes, treating timeouts as completed response attempts.
- Since a single node propagates the request to the other nodes, the communication cost of this phase is \(O(n)\).

### Prepare

```text
Prepare(
    proposeId,
    sentRequest,
    isRequestSignatureValid,
    signature
) -> (
    sentRequest,
    isRequestSignatureValid,
    signature
)
```

- In this phase, each node exchanges the value it received from the proposer and whether the corresponding client signature is valid.
- After this phase, each node has its own received value and information about the values observed by other nodes.
- Each claim also carries a signature as evidence that the corresponding node generated the claim. This signature serves as evidence that the node produced the value.
- Each node must generate exactly one such claim per consensus instance. Generating more than one conflicting claim causes the node to be considered Byzantine.
- Signatures are always verified when received. If verification fails, the corresponding observation is recorded as `Unknown`. The `Unknown` value does not require a signature and does not support any candidate in the final vote.
- For an unknown proposal, a Prepare request always returns `Unknown`.
- The communication cost is \(n(n+1)/2\), resulting in \(O(n^2)\) communication complexity.

### Merge

```text
Merge(
    proposeId,
    {
        clientId: [
            request,
            isRequestSignatureValid,
            prepareSignature
        ]
    },
    ...
) -> (
    {
        nodeId: {
            clientId: [
                request,
                isRequestSignatureValid,
                prepareSignature
            ],
            ...
        },
        ...
    }
)
```

- In this phase, nodes exchange the values they have observed.
- Each node includes its own observations. The observations of all nodes are exchanged.
- A single Prepare observation is insufficient to guarantee safety because a Byzantine node may equivocate. This issue is addressed by comparing the exchanged observations.
- This can be interpreted as an increase in the dimensionality of the information associated with each node's value.
- A Byzantine node may equivocate during this phase and send different information to different nodes. However, such behavior does not affect the final result.
- If a Merge value cannot be received because of silence, a timeout, or a similar event, it is treated as `Unknown`.
- At this point, all honest nodes share the same state. Prepare-phase equivocation is addressed through Merge, while Merge-phase equivocation is addressed through certificates.
- The communication cost is \(n(n+1)/2\), resulting in \(O(n^2)\) communication complexity.

### Conflict

- Conceptually, this phase is part of Merge, but it is separated for convenience.
- This phase begins after all Merge exchanges have completed.
- Based on the Merge results, each node determines the final claim associated with each node. This computation is performed locally.
- A conflicting claim may carry a signature. If a Byzantine node has forged a signature, that signature will fail verification.
- If two or more signatures associated with the same node are all valid but refer to different values, the node must have equivocated. The node is therefore treated as Byzantine, and its final claim is set to `Unknown`.
- Since the signed evidence observed by honest nodes is shared during Merge, if a Byzantine node sends conflicting signed values to honest nodes, all honest nodes can identify the equivocation.
- For a node for which equivocation has not been detected, its final claim is set to a value reported by at least a strict majority (\(f+1\) or more) of the valid signed observations received during Merge. If no value reaches this threshold, the node's final claim is set to `Unknown`.
- Even if a Byzantine node equivocates during Merge, at most \(f\) Byzantine nodes can interfere. Therefore, they cannot corrupt the honest nodes' claims or cause different honest nodes to adopt conflicting claims.
- After the final claims associated with all nodes have been determined, the candidate value supported by a strict majority of at least \(f+1\) claims is selected as the final Conflict result. If no candidate reaches this threshold, the result is `Reject`.

### Commit

- The final commit decision is made based on the result of the Conflict phase.
- If a node's claim is set to `Unknown` because of signature verification failure, a timeout, silence, equivocation, or a similar event, that node is treated as not supporting any candidate value.
- Regardless of whether `Unknown` occurs, the total quorum threshold \(Q=f+1\) and the fault bound \(f\) remain fixed and are not reduced.
- If Conflict produces a valid candidate value rather than `Reject`, each honest node signs that value and propagates its signature to the other nodes. If the result is `Reject`, no Commit signature is generated by an honest node.
- All honest nodes either derive the same valid candidate value or derive `Reject`. Since there are at most \(f\) Byzantine nodes, they cannot independently obtain the \(f+1\) signatures required to commit a value, and conflicting values cannot both be committed.
- A collection of valid Commit signatures from at least \(f+1\) distinct nodes for the same value constitutes a valid Commit certificate.
- Honest nodes do not sign two different values for the same consensus instance. Since Byzantine nodes alone cannot obtain \(f+1\) signatures, two conflicting Commit certificates cannot coexist.
- The client can determine whether a value has been validly committed by obtaining the \(f+1\) signatures that constitute its Commit certificate.
- Since nodes exchange their signatures with one another, the communication complexity is \(O(n^2)\).

---

# MBQ Lemmas

## State Definitions and Notation

The following states have distinct meanings.

- **`Unknown`**: A node state indicating that the node cannot support a particular candidate because it lacks a required response or valid evidence, such as in cases of timeout, silence, signature verification failure, or detected equivocation. `Unknown` does not support any candidate and does not, by itself, establish that the node is faulty. Regardless of whether `Unknown` occurs, the global quorum threshold \(Q=f+1\) is not reduced.
- **`Reject`**: A local decision produced by the Conflict phase when no candidate obtains a strict majority of \(Q=f+1\). An honest node that returns `Reject` does not generate a Commit signature.

Common notation:

- \(n=2f+1\): The size of the complete node set \(V\), where \(|V|=2f+1\).
- \(f\): The upper bound on the number of faulty nodes, including Byzantine, crash, and omission faults.
- \(Q=f+1\): The strict-majority quorum size.
- \(H\): The set of honest nodes, where \(|H|\ge f+1\).
- \(F\): The set of faulty nodes, where \(|F|\le f\), \(H\cap F=\emptyset\), and \(H\cup F=V\).
- \(E_k(v)=(v,\sigma_k(v))\): The pair consisting of node \(k\)'s Prepare claim for candidate value \(v\) and its valid node signature.
- \(M_i\): The set of all signed evidence and observation reports collected by honest node \(i\in H\) by the Merge deadline.
- \(c_i(k)\in\mathcal{V}\cup\{\text{Unknown}\}\): The final Prepare claim for node \(k\in V\), as determined by honest node \(i\in H\) based on \(M_i\), where \(\mathcal{V}\) is the set of valid candidate values.
- \(C_i\in\mathcal{V}\cup\{\text{Reject}\}\): The final Conflict result derived by honest node \(i\in H\).

Each node is counted as a valid supporter of at most one candidate value per consensus instance. Conflicting claims from the same node are not counted as independent support for multiple candidate values.

---

## Lemma A — Detection of Equivocation Through Conflicting Signatures

**Claim.** If the same node generates valid signatures for two conflicting claims in the same consensus instance and both signatures are made available to a verifier, the node's equivocation can be detected deterministically.

**Proof.**

Suppose node \(b\in V\) generates valid signatures \(\sigma_b(v_1)\) and \(\sigma_b(v_2)\) for two distinct candidate values \(v_1,v_2\in\mathcal{V}\) in the same consensus instance, where \(v_1\ne v_2\).

$$
v_1\ne v_2
$$

The signatures are unforgeable and both verify under the public key of the same node \(b\). Since an honest node generates a signature for only one value in a given consensus instance, the existence of the pair of valid signatures \((\sigma_b(v_1),\sigma_b(v_2))\) for two different values establishes that node \(b\in F\) has equivocated.

Therefore, a verifier that possesses both signatures can compare them, deterministically detect the equivocation, and invalidate the node's claim by setting it to `Unknown`.

**Conclusion.** Once both conflicting valid signatures are available, the corresponding equivocation can be detected deterministically.

---

## Lemma B — Evidence Sharing and Compatibility of Node Claims After Merge

**Claim.** Suppose all honest nodes broadcast during Merge the sets of valid signed evidence that they generated or observed during Prepare, and messages between honest nodes are delivered by the established deadline. Each honest node \(i\in H\) determines the final claim \(c_i(k)\) of every node \(k\in V\) according to the following rules:

1. **Equivocation invalidation rule:** If \(M_i\) contains two valid conflicting signatures \(\sigma_k(v_1)\) and \(\sigma_k(v_2)\) from node \(k\), where \(v_1\ne v_2\), then \(c_i(k)=\text{Unknown}\).
2. **Merge majority rule:** If no conflicting signatures exist and at least \(Q=f+1\) distinct nodes report the same valid signed evidence \(E_k(v)\) during Merge, then and only then \(c_i(k)=v\). Otherwise, \(c_i(k)=\text{Unknown}\).

Under these conditions, it is impossible for two different honest nodes \(i,j\in H\) to simultaneously finalize conflicting candidate values \(v_1\ne v_2\) for the same node \(k\).

**Proof.**

Suppose an arbitrary honest node \(p\in H\) observes valid signed evidence \(E_k(v)=(v,\sigma_k(v))\) from node \(k\) during Prepare. Node \(p\) broadcasts this evidence during Merge.

By the synchronous network assumption, the Merge message sent by \(p\) is delivered to every honest node before the deadline. Therefore, the following inclusion property holds:

$$
\forall p,i\in H,\qquad
E_k(v)\in M_p\implies E_k(v)\in M_i
$$

We proceed by contradiction. Assume that two honest nodes \(i,j\in H\) finalize two different candidate values \(v_1,v_2\in\mathcal{V}\), where \(v_1\ne v_2\), for the same node \(k\in V\):

$$
c_i(k)=v_1,\qquad
c_j(k)=v_2
\qquad (v_1\ne v_2)
$$

By the Merge majority rule, node \(i\) must have received reports of \(E_k(v_1)\) from at least \(f+1\) distinct nodes, and node \(j\) must have received reports of \(E_k(v_2)\) from at least \(f+1\) distinct nodes.

Since the number of faulty nodes is bounded by \(f\), with \(|F|\le f\), at least one honest node \(p\in H\) must exist among the \(f+1\) or more nodes reporting \(E_k(v_1)\). Similarly, at least one honest node \(q\in H\) must exist among the \(f+1\) or more nodes reporting \(E_k(v_2)\).

The evidence \(E_k(v_1)\) and \(E_k(v_2)\) broadcast by the honest nodes \(p\) and \(q\), respectively, is delivered to every honest node before the Merge deadline. Consequently, both evidence sets \(M_i\) and \(M_j\) contain both signed claims:

$$
\{E_k(v_1),E_k(v_2)\}\subseteq M_i\cap M_j
$$

By Lemma A and the equivocation invalidation rule, both nodes \(i\) and \(j\) must therefore set their claims to `Unknown`, such that \(c_i(k)=\text{Unknown}\) and \(c_j(k)=\text{Unknown}\). This contradicts the assumption.

**Conclusion.** For every node \(k\in V\), there exists at most one candidate value \(v_k^*\in\mathcal{V}\) such that, for all honest nodes \(i\in H\),

$$
\forall i\in H,\qquad
c_i(k)\in\{v_k^*,\text{Unknown}\}
$$

In particular, an honest node \(k\in H\) does not equivocate and distributes its own Prepare evidence \(E_k(P_k)\) to all honest nodes. Therefore, every honest node \(i\in H\) receives at least \(|H|\ge f+1\) matching reports and finalizes \(c_i(k)=P_k\).

---

## Lemma C — Quorum Limit of Faulty Nodes

**Claim.** If there are at most \(f\) faulty nodes and both the majority quorum and the Commit certificate require support or signatures from \(Q=f+1\) distinct nodes, faulty nodes alone cannot satisfy the quorum or construct a Commit certificate.

**Proof.**

The number of faulty nodes and the quorum size satisfy

$$
|F|\le f<f+1=Q
$$

Therefore, faulty nodes alone cannot obtain support or signatures from \(f+1\) distinct nodes.

`Unknown` does not count as support for any candidate value. Likewise, conflicting claims from the same node are not counted as independent support for multiple candidates. Equivocation is detected as described in Lemma A.

The occurrence of `Unknown` does not itself establish that a node is faulty. Neither `Unknown` nor detected equivocation reduces \(f\) or changes the quorum threshold \(Q=f+1\).

**Conclusion.** Claims or signatures from faulty nodes alone cannot satisfy the majority quorum in Conflict or the certificate threshold in Commit.

---

## Lemma D — Non-Contradiction of Conflict Results

**Claim.** If every honest node applies the strict-majority decision rule, with \(Q=f+1\), to the per-node claims established by Lemma B, then two different honest nodes \(i,j\in H\) cannot derive two conflicting candidate values \(v_1\ne v_2\), where \(v_1,v_2\in\mathcal{V}\), as their respective Conflict results.

**Proof.**

Let the set of nodes whose claims are finalized as supporting candidate value \(v\in\mathcal{V}\) by honest node \(i\in H\) be defined as

$$
S_i(v)=\{k\in V\mid c_i(k)=v\}
$$

The Conflict decision rule returns \(C_i=v\) when a candidate \(v\) satisfies \(|S_i(v)|\ge f+1\). If no candidate satisfies this condition, it returns \(C_i=\text{Reject}\).

Proceeding by contradiction, suppose two honest nodes \(i,j\in H\) derive two distinct valid candidate values \(v_1,v_2\in\mathcal{V}\), where \(v_1\ne v_2\), such that \(C_i=v_1\) and \(C_j=v_2\).

By the quorum condition, the two support sets satisfy

$$
|S_i(v_1)|\ge f+1,\qquad
|S_j(v_2)|\ge f+1
$$

Since the total number of nodes is \(|V|=2f+1\), the inclusion-exclusion principle gives the following lower bound on the intersection of the two support sets:

$$
\begin{aligned}
|S_i(v_1)\cap S_j(v_2)|
&=|S_i(v_1)|+|S_j(v_2)|\\
&\quad-|S_i(v_1)\cup S_j(v_2)|\\
&\ge(f+1)+(f+1)-(2f+1)\\
&=1
\end{aligned}
$$

Therefore, at least one node \(k\in V\) belongs to the intersection \(S_i(v_1)\cap S_j(v_2)\). This node must satisfy

$$
c_i(k)=v_1
\quad\text{and}\quad
c_j(k)=v_2
$$

However, by Lemma B, two honest nodes \(i,j\) cannot simultaneously finalize different candidate values \(v_1\ne v_2\) for the same node \(k\). This is a contradiction.

**Conclusion.** Within the same consensus instance, there is at most one valid candidate value \(v^*\in\mathcal{V}\) that any honest node can derive as its Conflict result. Specifically,

$$
\forall i\in H,\qquad
C_i\in\{v^*,\text{Reject}\}
$$

In other words, different honest nodes cannot derive different valid candidate values.

---

## Lemma E — Safety of Commit Certificates

**Claim.** Suppose every honest node \(i\in H\) generates a Commit signature for its Conflict result only when that result is a valid candidate value, i.e., \(C_i\ne\text{Reject}\). Then two valid Commit certificates for different values \(v_1\ne v_2\) cannot simultaneously exist within the same consensus instance.

**Proof.**

By Lemma D, at most one valid candidate value \(v^*\in\mathcal{V}\) can be derived by honest nodes in a given consensus instance, excluding `Reject`.

Each honest node generates a Commit signature only for its own valid result. Therefore, no honest node generates a Commit signature for a conflicting value \(v'\ne v^*\), where \(v'\in\mathcal{V}\).

A certificate for \(v'\) requires signatures from at least \(f+1\) distinct nodes. Since no honest node signs \(v'\), only faulty nodes can provide such signatures.

By Lemma C, the number of available signatures satisfies

$$
\#\operatorname{Signers}(v')
\le |F|
\le f
< f+1=Q
$$

This contradicts the Commit certificate requirement.

**Conclusion.** Two valid Commit certificates for different values cannot simultaneously exist within the same consensus instance.

A Commit certificate includes the client request, client signature, Commit signatures, logical time, and other relevant information. Its specific fields and validation rules are defined in the protocol specification.

---

## Lemma F — Safety Against a Byzantine Proposer

**Claim.** Even if a Byzantine proposer selectively delivers a request to only some nodes, sends conflicting requests to different nodes, or stops forwarding the request, it cannot cause Commit certificates for different values to be generated under the conditions of the preceding lemmas.

**Proof.**

A valid client signature allows the request contents to be verified. Therefore, a proposer cannot arbitrarily modify the contents of a signed request without forging the client's signature.

If the proposer selectively delivers a request or equivocates, some honest nodes may lack evidence of the request or may observe conflicting evidence. However, the Prepare results generated and observed by honest nodes are shared during Merge under the conditions of Lemma B.

Consequently, by Lemma D, the Conflict results of all honest nodes are restricted to a single candidate value \(v^*\) and `Reject`, such that

$$
C_i\in\{v^*,\text{Reject}\}
$$

By Lemma E, no Commit certificate can be generated for another value \(v'\ne v^*\).

**Conclusion.** Under the conditions established in the preceding lemmas, selective delivery, equivocation, or silence by a Byzantine proposer cannot violate Commit safety.

This lemma concerns safety. Progress through retries is addressed separately in Lemma G.

---

## Lemma G — Validity and Probabilistic Progress Through Honest Proposer Selection

**Claim.** When an honest proposer is selected for a consensus instance, a valid Commit certificate is guaranteed to be generated for a valid client request value \(v\) (validity). Furthermore, if the proposer is selected uniformly and independently from all \(n=2f+1\) nodes on each retry, and at most \(f\) nodes are faulty, the probability of selecting an honest proposer in a single attempt satisfies

$$
p\ge\frac{f+1}{2f+1}>\frac12
$$

**Proof.**

Suppose the selected proposer is honest, that is, it belongs to \(H\). It forwards the client's valid request \(v\) to all honest nodes before the deadline.

Every honest node \(k\in H\) generates and shares the same Prepare evidence \(E_k(v)\). By Lemma B, every honest node \(i\in H\) therefore finalizes \(c_i(k)=v\) for every \(k\in H\).

Since the number of honest nodes satisfies \(|H|\ge f+1=Q\), the support set for \(v\) at every honest node \(i\in H\) satisfies

$$
|S_i(v)|\ge f+1
$$

Consequently, every honest node derives \(C_i=v\). At least \(f+1\) honest nodes therefore generate Commit signatures for \(v\), and a valid Commit certificate is deterministically generated.

Since the total number of nodes is \(2f+1\) and at most \(f\) nodes are faulty, the probability of selecting an honest proposer under uniform random selection is

$$
p=\frac{|H|}{2f+1}
\ge\frac{f+1}{2f+1}
>\frac12
$$

If each attempt is independent, the probability that an honest proposer is selected at least once within \(k\) attempts is bounded below by

$$
P(\text{at least one honest proposer})
\ge 1-\left(\frac{f}{2f+1}\right)^k
$$

Therefore, as \(k\) increases, the probability of selecting an honest proposer at least once converges to \(1\).

**Conclusion.** When an honest proposer is selected, consensus is reached within a single round under the stated assumptions. Under a uniformly random proposer-selection policy, the probability of selecting an honest proposer exceeds one half in each attempt, allowing the probability of selecting an honest proposer through independent retries to be made arbitrarily high.