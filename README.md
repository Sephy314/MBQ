# MBQ: Merge-Based and Majority-Quorum BFT Protocol

## Purpose

- This protocol aims to achieve safety in a Byzantine environment using only a majority quorum in a consensus algorithm. It addresses Byzantine consensus through evidence exchange, unforgeable signatures, and the assumption of synchronous communication.

## Assumptions

- Byzantine nodes may exhibit crashes, network failures, timeouts, partial or complete silence, equivocation, false requests, collusion, and other Byzantine behaviours.
- The communication environment is assumed to be synchronous.
- This algorithm addresses only the consensus problem. Failure recovery is the responsibility of the implementation.
- The system consists of \(2f+1\) nodes, of which at most \(f\) may be Byzantine, crash-faulty, or omission-faulty, while \(f+1\) nodes are honest.
- Even a legitimate client request may fail to be delivered under certain circumstances, but it can succeed through retries.
- The retry policy is the responsibility of the client.
- Clients are always honest and are assumed never to become Byzantine.
- All nodes and clients can generate unforgeable digital signatures.
- Load-balancing policies that permanently associate a client with a single server, such as IP hashing, are unsuitable for this approach.
- `Unknown` is independent of whether a node is Byzantine.
- Signatures are unforgeable.
- The same client request is committed at most once.
- Client retries are the responsibility of the implementation. Idempotency must be preserved, for example, by checking the log for an existing instance of the same request.
- False failures may occur. The policy for handling them is the client's responsibility. A false failure initiates a new consensus instance that is not associated with any previous consensus instance.
- Each signature covers all the contents of its corresponding message. A signature cannot be reused across different phases or requests.
- A phase terminates when responses have been received from all nodes or a timeout occurs. Messages arriving after phase termination are not accepted.
- Each consensus instance handles exactly one client request.
- Commit-certificate signatures may be optimised using techniques such as threshold signatures, provided that safety is preserved, depending on the implementation.
- A node signs at most one final claim for a given consensus instance.

## Phases

### Propose

```text
Propose(clientId, request, requestSignature) -> (proposeId)
```

- The node that initiates the proposal becomes the proposer for the current consensus instance.
- This phase forwards the request received from the client to the other nodes.
- The proposer advances to the next phase after receiving responses from all nodes, including timeout outcomes.
- Since the proposer propagates the request to all nodes, the time complexity of this phase is \(O(n)\).

### Prepare

```text
Prepare(proposeId, sentRequest, isRequestSignatureValid, Signature)
    -> (sentRequest, isRequestSignatureValid, Signature)
```

- In this phase, each node exchanges the value it received from the proposer and whether the associated signature is valid.
- After this phase, each node possesses its own observation and the observations reported by other nodes.
- Each report includes a signature that serves as evidence that the corresponding node made the claim.
- Every node must generate exactly one valid signature per consensus instance. Generating more than one valid signature for the same instance is considered equivocation.
- Signatures are always verified upon receipt. If a node did not receive the `Propose` message but receives a `Prepare` message containing a valid client signature, it joins the consensus instance. Its own claim is `Unknown`.
- Nodes that did not receive `Propose` participate in subsequent phases if they receive a `Prepare` message containing a valid client signature.
- A response to a `Prepare` message concerning an unknown proposal is always `Unknown`.
- The time complexity is \(O(n^2)\).

### Merge

```text
Merge(proposeId, {clientId: [request, isRequestSignatureValid, prepareSignature]}, ...)
    -> ({nodeId: {clientId: [request, isRequestSignatureValid, prepareSignature]}, ...})
```

- This phase exchanges each node's observations with the other nodes.
- Each node includes its own observations. The observations of all nodes and their corresponding signatures, which serve as evidence, are exchanged.
- A single `Prepare` report does not guarantee safety because a Byzantine node may equivocate. This problem is addressed by comparing the observed values with one another.
- This can be viewed as increasing the dimensionality of the information associated with each node's claims.
- A Byzantine node may equivocate during this phase and send different information to different recipients. However, this does not affect the final safety guarantee.
- If a Merge value cannot be received because of silence, a timeout, or a similar failure, it is treated as `Unknown`.
- Claims made by honest nodes during `Prepare` are shared identically with all honest nodes through `Merge`. Equivocation during `Prepare` is addressed through equivocation detection in the `Merge` and `Conflict` phases.
- Every honest node exchanges all of its observed `Prepare` evidence with every other honest node, without omitting any such evidence.
- The time complexity is \(O(n^2)\). To reduce bit complexity, an implementation may introduce a hash-verification fast path. Since this does not change the logical protocol, it is left to the implementation.

### Conflict

- This phase is conceptually a subphase of `Merge`, but is separated for clarity.
- It begins once the exchange of values resulting from `Merge` has completed.
- Each node locally consolidates the final claims of all nodes based on the Merge results.
- A conflicting claim may carry a signature. If the claim is malicious and was generated by a Byzantine node, the signature may be invalid.
- If two or more signatures associated with the same node support different values and all of those signatures are valid, the only remaining possibility is that the node equivocated. The node is therefore treated as Byzantine, and its final claim is set to `Unknown`.
- Since honest nodes exchange observed signature evidence during `Merge`, all honest nodes can determine that a node equivocated if a Byzantine node sent different signed values to different honest nodes.
- In addition, if a node's claim about itself in `Merge` differs from its own reported value in `Prepare`, the inconsistency can be identified as equivocation.
- For each node that passes equivocation detection, a claim containing a valid signature is accepted as that node's final claim. If no valid signature exists, the node's final claim is assumed to be `Unknown`.
- Even if Byzantine nodes equivocate during `Merge`, at most \(f\) Byzantine nodes can interfere with the process. They cannot corrupt honest nodes' claims or cause different honest nodes to adopt conflicting claims.
- Once the final claims of all nodes have been consolidated, the candidate value supported by a majority quorum of \(f+1\) nodes is selected as the final `Conflict` result. If no candidate value exceeds the majority threshold, the result is `Reject`.

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

- The final commit decision is determined based on the result of the `Conflict` phase.
- If a node's claim is treated as `Unknown` because of signature verification failure, a timeout, non-response, equivocation, or a similar condition, that node is considered not to have voted for any candidate value.
- However, the occurrence of `Unknown` does not reduce the overall quorum threshold \(Q=f+1\) or the value of \(f\). Both remain fixed.
- If the `Conflict` result is a valid candidate value other than `Reject`, honest nodes sign that value and propagate their signatures to one another. If the result is `Reject`, they do not sign it.
- All honest nodes derive the same valid candidate value or derive `Reject`. Since there are at most \(f\) Byzantine nodes, they cannot exceed the majority threshold \(f+1\) on their own and therefore cannot commit conflicting values.
- Each node signs the information required to identify the client request and shared across the consensus instance, including `proposeId`, `clientRequest`, and `clientSignature`, using its private key, and provides the resulting signature.
- A receiving node verifies the validity of the signature.
- If at least \(f+1\) valid commit signatures from distinct nodes are collected for a particular value, the result is recognised as a valid commit certificate.
- There is only one candidate value per consensus instance. Since at most \(f\) nodes are Byzantine, they cannot independently reach the quorum of \(f+1\) signatures; therefore, safety is preserved.
- Honest nodes do not sign different values, and Byzantine nodes alone cannot obtain \(f+1\) signatures. Consequently, two conflicting commit certificates cannot coexist.
- The client determines whether the value has been successfully committed by obtaining \(f+1\) commit signatures forming a commit certificate.
- Since nodes exchange their signatures with one another, the communication complexity is \(O(n^2)\).

# MBQ Lemmas

## Lemma A — Detection of Conflicting Claims Through Evidence Exchange

**Statement.** If a Byzantine node generates valid signatures supporting different values for the same claim target within the same consensus instance, and those signatures are delivered to honest nodes through evidence exchange during the `Merge` phase, the honest nodes can detect the conflicting claims.

**Proof.**

Suppose that a Byzantine node \(b\) generates two valid signatures, \(\sigma_b(v_1)\) and \(\sigma_b(v_2)\), for two distinct values \(v_1 \ne v_2\). If both signatures belong to the same consensus instance and the same claim target, they represent mutually incompatible claims.

By the unforgeability of signatures, both signatures were genuinely generated by node \(b\). Therefore, if an honest node observes both signatures and verifies that they are mutually conflicting, it can detect the conflicting claims made by node \(b\).

Even if different honest nodes observe different signatures during `Prepare`, they can jointly identify the conflicting signatures by exchanging their observed evidence during `Merge`. Furthermore, if a node's own `Prepare` claim differs from the claim it reports for itself in `Merge`, the inconsistency is treated as a conflicting claim, and that node's claim is marked as `Unknown`.

Therefore, whenever evidence of conflicting signatures is delivered to an honest node, the conflicting claims can be detected. \(\square\)

## Lemma B — Safety Against Silence and Selective Responses

**Statement.** The silence or selective responses of Byzantine nodes may impede consensus progress or prevent the acquisition of sufficient evidence for commitment. However, silence itself cannot constitute valid evidence supporting a value, nor can it reduce the required quorum.

**Proof.**

We distinguish the following cases of Byzantine silence.

**Case 1: Silence during `Propose`.**

An honest node that receives a valid request and client signature forwards them during `Prepare`. A node that does not receive the `Propose` message may participate according to the protocol with its own claim set to `Unknown`. Therefore, selective delivery during `Propose` may cause nodes to observe different information, but it does not change the criteria for determining request validity.

**Case 2: Complete silence during `Prepare`.**

If a node's claim is not delivered to any honest node, the honest nodes possess no valid supporting evidence for that claim. The claim is treated as `Unknown` and is not counted as support for any valid candidate value.

The number of Byzantine nodes is at most \(f\), and the required quorum is \(Q=f+1\). Therefore, Byzantine claims alone cannot satisfy the required quorum.

**Case 3: Selective silence during `Prepare`.**

If a Byzantine node's claim is delivered to at least one honest node, that honest node exchanges the observed evidence with the other honest nodes during `Merge`. Under the synchronous communication assumption and honest protocol compliance, the evidence is shared among the honest nodes.

**Case 4: Silence during `Merge`.**

Even if a Byzantine node sends no messages during `Merge`, if its claim was observed by at least one honest node during `Prepare`, that evidence is propagated to the other honest nodes through the corresponding honest node's `Merge` message. Therefore, silence during `Merge` alone cannot remove evidence that has already been observed from the exchange among honest nodes.

Conversely, if the claim was not observed by any honest node, honest nodes treat it as `Unknown` and do not count it as valid support.

In all of the above cases, silence may lead to `Reject` or prevent commitment. However, silence itself does not constitute valid supporting evidence and does not reduce the required quorum \(Q=f+1\). \(\square\)

## Lemma C — Safety Against a Byzantine Proposer

**Statement.** A Byzantine proposer can impede consensus progress through request omission, invalid client signatures, or selective request delivery. However, provided that honest nodes verify request validity and comply with the commit rules, the Byzantine proposer's actions alone cannot cause an invalid request to be committed.

**Proof.**

We consider the following cases of Byzantine proposer behaviour.

**Case 1: The request or client signature is missing or invalid.**

Honest nodes do not recognise the request as a valid candidate. If valid request evidence and the required support cannot be obtained, the `Conflict` phase produces `Reject`, and honest nodes do not generate commit signatures for that request.

**Case 2: The request is delivered to only some honest nodes.**

Honest nodes that receive the request forward it together with the client signature during `Prepare`. Honest nodes that do not receive the request may participate with their claims set to `Unknown`. This may prevent sufficient support from being established, but it does not change the rules for determining request validity.

**Case 3: Byzantine nodes cooperate to obstruct request delivery.**

There are at most \(f\) Byzantine nodes. Their signatures alone cannot provide \(Q=f+1\) signatures from distinct nodes. If honest support does not reach the required quorum, the consensus instance may terminate with `Reject`.

Conversely, even if a commit occurs, the commit certificate must satisfy the protocol's signature requirements and contain a valid request. Therefore, a Byzantine proposer may impede consensus progress, but it cannot bypass honest nodes' verification rules to commit an invalid request. \(\square\)

## Lemma D — Probability of Electing an Honest Proposer Through Independent Retries

**Statement.** If a proposer is selected independently and uniformly at random for each retry among \(n=2f+1\) nodes, the probability that at least one honest proposer is selected within \(k\) retries is given by

\[
\Pr[\text{honest proposer elected}]
\geq
1-\left(\frac{f}{2f+1}\right)^k.
\]

Consequently, as \(k\to\infty\), this probability converges to \(1\).

**Proof.**

There are at most \(f\) Byzantine nodes. Therefore, the probability of selecting a Byzantine proposer in a single independent draw is at most \(f/(2f+1)\).

Consequently, the probability that every proposer selected across \(k\) independent draws is Byzantine is at most

\[
\left(\frac{f}{2f+1}\right)^k.
\]

The complementary event is that an honest proposer is selected at least once. Therefore,

\[
\Pr[\text{honest proposer elected}]
\geq
1-\left(\frac{f}{2f+1}\right)^k.
\]

Since \(f/(2f+1)<1\), the right-hand side converges to \(1\) as \(k\) increases. \(\square\)

**Interpretation.** This lemma directly establishes the probability of electing an honest proposer. A separate liveness proof is required to establish that commitment is guaranteed after an honest proposer has been selected.


## Lemma E — Consistent Final Interpretation of Honest-Node Claims

**Statement.** If an honest node \(h\) generates and signs a valid claim \(x_h\) during `Prepare` within a given consensus instance, then, under the synchronous communication assumption and honest protocol compliance, that evidence is delivered to all honest nodes through `Merge`. Consequently, every honest node interprets the claim identically during `Conflict`.

That is,

\[
\forall h,i\in H,\quad
\mathrm{FinalClaim}_i(h)=x_h
\]

**Proof.**

An honest node \(h\) generates a valid signature \(\sigma_h(x_h)\) for its claim \(x_h\) during `Prepare`. By signature unforgeability, no other node can forge or modify that signature.

The honest node reports its own claim during `Prepare`. Under the synchronous communication assumption and honest protocol compliance, the evidence is delivered to the other honest nodes. Subsequently, during `Merge`, each honest node exchanges its observed `Prepare` evidence with the other honest nodes. Therefore, the valid signature evidence generated by \(h\) is shared with every honest node.

Byzantine nodes may selectively deliver their own claims or collude with other Byzantine nodes. However, these actions cannot forge honest nodes' signatures or alter honest-node evidence that has already been shared.

Furthermore, if a valid signature generated by an honest node \(h\) had a conflicting valid signature, this would contradict signature unforgeability and honest protocol compliance. Therefore, the claim is not invalidated on the grounds of equivocation.

Since all honest nodes possess the same signature evidence and apply the same verification and `Conflict` decision rules, they accept the claim as the same final claim.

\[
\forall h,i\in H,\quad
\mathrm{FinalClaim}_i(h)=x_h
\]

Therefore, the statement holds. \(\square\)

## Lemma F — Agreement on Valid Conflict Results Among Honest Nodes

**Statement.** Even if Byzantine nodes equivocate or respond selectively during `Merge`, two different honest nodes cannot derive different valid candidate values (i.e., values other than `Reject`) as their final results during `Conflict`.

\[
\forall i,j\in H,\quad
(C_i\ne\text{Reject}\land C_j\ne\text{Reject})
\Rightarrow C_i=C_j
\]

**Proof.**

Partition the set of all nodes into the set of honest nodes \(H\), where \(|H|=f+1\), and the set of Byzantine nodes \(B\), where \(|B|\leq f\).

1. **Consistency of honest-node claims.**

   Each honest node \(h\in H\) generates a claim during `Prepare` and signs it with an unforgeable signature. Under the synchronous communication assumption and honest protocol compliance, the claim is shared identically with all honest nodes during `Merge`. Byzantine nodes cannot forge honest nodes' signatures. Therefore, all honest nodes \(i,j\in H\) derive exactly the same final claims for nodes belonging to \(H\).

2. **Limitations of Byzantine claims and the majority quorum.**

   Even if Byzantine nodes equivocate or remain silent during `Merge`, any resulting differences in the observed claims are confined to the claims of the at most \(f\) Byzantine nodes in \(B\). (However, if a Byzantine node sends conflicting signatures to two different honest nodes during `Prepare`, Lemma A ensures that all honest nodes treat that node's claim as `Unknown`.)

   Suppose, for contradiction, that two honest nodes \(i,j\in H\) derive different valid candidate values \(v_1\ne v_2\), where \(v_1,v_2\ne\text{Reject}\).

   According to the `Conflict` decision rule, a valid candidate value must be supported by at least the majority quorum \(Q=f+1\). Since there are at most \(f\) Byzantine nodes, at least one node in \(H\) must support \(v_1\) for node \(i\) to derive \(v_1\).

   The final claims of honest nodes are observed identically by every honest node. Let \(k\) be the number of honest nodes in \(H\) that support \(v_1\), where \(k\geq1\). Node \(j\) must therefore count the same \(k\) honest nodes as supporting \(v_1\).

   Consequently, at most \((f+1)-k\leq f\) honest nodes remain available to support a different value \(v_2\).

   Furthermore, within a single consensus instance, the valid client-signed request is unique. Thus, honest nodes cannot simultaneously support two different valid values \(v_1\ne v_2\). Even if Byzantine nodes concentrate their votes on \(v_2\), they can contribute at most \(f\) votes. If Byzantine nodes send conflicting signatures supporting both \(v_1\) and \(v_2\) to honest nodes, Lemma A ensures that their claims are treated as `Unknown` and are not counted as votes.

   Consequently, it is impossible for two different valid candidate values \(v_1\ne v_2\) to simultaneously obtain support of at least \(Q=f+1\) at two honest nodes.

   Therefore, if \(C_i\ne\text{Reject}\) and \(C_j\ne\text{Reject}\), then \(C_i=C_j\). \(\square\)

## Lemma G — Uniqueness of Committed Values

**Statement.** Suppose that Lemma E holds, that honest nodes sign a value only when their final `Conflict` result is valid (i.e., not `Reject`), and that each honest node generates valid commit signatures for at most one value within the same consensus instance. Then two valid commit certificates for different values cannot coexist.

That is,

\[
Cert(v_1)\land Cert(v_2)\Rightarrow v_1=v_2
\]

**Proof.**

Assume, for contradiction, that two valid commit certificates \(Cert(v_1)\) and \(Cert(v_2)\) exist simultaneously for distinct values \(v_1\ne v_2\).

Each certificate contains valid signatures from \(Q=f+1\) distinct nodes. Since there are at most \(f\) Byzantine nodes, each certificate must contain at least one honest node's signature.

Let \(h_1,h_2\in H\) be honest nodes that signed \(v_1\) and \(v_2\), respectively. Since honest nodes sign a value only when their final `Conflict` result is valid (i.e., not `Reject`),

\[
C_{h_1}=v_1\ne\text{Reject},
\qquad
C_{h_2}=v_2\ne\text{Reject}.
\]

However, by Lemma E, if the `Conflict` results of two honest nodes are both different from `Reject`, those results must be identical:

\[
C_{h_1}=C_{h_2}.
\]

It follows that \(v_1=v_2\), contradicting the initial assumption that \(v_1\ne v_2\).

Therefore, two valid commit certificates for different values cannot coexist within the same consensus instance. \(\square\)
