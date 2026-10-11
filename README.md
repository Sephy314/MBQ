# MBQ: Merge-Based and Majority Quorum BFT Protocol

## Purpose

* This protocol aims to achieve safety in Byzantine environments using majority quorums in a consensus algorithm and to interpret Byzantine consensus from a set-theoretic perspective.

## Assumptions

* Byzantine nodes may crash, remain partially or completely silent, equivocate, submit false requests, collude, and perform other Byzantine behaviors.
* The communication environment is assumed to be synchronous.
* There are \(f\) Byzantine nodes and \(f+1\) honest nodes, for a total of \(2f+1\) nodes.
* Honest nodes always produce the same value.
* The client is assumed to always be honest and cannot be Byzantine.
* All nodes and the client can generate unforgeable digital signatures.

## Features

* A quorum of \(f+1\) is required.
* Nodes confirmed to be Byzantine are expelled from the system.

## Phases

### Propose

```
Propose(clientId, request, requestSignature) -> (proposeId)
```

* The node that initiates the proposal becomes the proposer for the current consensus instance.
* In this phase, the proposer forwards the request received from the client to the other nodes.
* The proposer advances to the next phase after receiving responses from all nodes, including timeout responses.
* Since a single node propagates the request to the other nodes, the time complexity is \(O(n)\).

### Prepare

```
Prepare(proposeId) -> (sentRequest, isRequestSignatureValid, signature)
```

* In this phase, each node returns the value it received from the proposer and whether the client's request signature is valid.
* After this phase, each node possesses its own observed value and information about values observed by other nodes.
* Each node also provides a signature as evidence of its claim, attesting to the value it reported.
* Each node must generate at most one such signature per consensus instance. Generating more than one conflicting signature is considered Byzantine behavior.
* Whenever a signature is received, it is verified. If verification fails, the corresponding value is recorded as `Unknown`. The `Unknown` value does not require a signature and does not support any candidate in the final vote.
* All nodes exchange these messages with one another. Therefore, the message complexity is \(O(n^2)\).

### Merge

```
Merge(proposeId) -> ({clientId: [request, isRequestSignatureValid]}, ...)
```

* In this phase, each node shares the values it observed with the other nodes.
* A Prepare phase alone cannot guarantee safety in the presence of Byzantine equivocation. This phase addresses the issue by comparing the observations reported by different nodes.
* This can be interpreted as increasing the dimensionality of the information associated with each node.
* Byzantine nodes may send different information to different recipients during Merge. However, such equivocation cannot affect the final value.
* After this phase, all honest nodes share the same state. Equivocation during Prepare is addressed through Merge, while equivocation during Merge is addressed through certificate verification.
* The message complexity is \(O(n^2)\).

### Conflict

* Each node locally determines its final claim based on the results of Merge.
* A value that obtains a majority among the values received during Merge is treated as that node's final claim.
* If no value obtains a majority, the final claim is designated as `Unknown`. `Unknown` is included in the total vote count but does not support any candidate value.
* Conflicting claims may be accompanied by signatures. If a malicious claim was fabricated by a Byzantine node, its signature may be invalid.
* If two or more signatures are valid but attest to conflicting claims from the same node, that node is considered Byzantine because such behavior is otherwise impossible under the protocol's assumptions. The node is then excluded from consideration.
* When a Byzantine node is removed, \(f\) is redefined as \(f-1\), and the subsequent process continues with the updated fault parameter.
* If a Byzantine node equivocates during Prepare, the resulting observations differ, allowing all nodes to identify the equivocation.
* Even if a Byzantine node equivocates during Merge, only the \(f\) Byzantine nodes can interfere with the process. They cannot corrupt the claims established by the honest nodes.
* Each node's final claim is determined only after the final equivocation-related processing has been completed.

### Kick
```
Kick(proposeId, nodeToBeKicked) -> (isAccepted)
```

* This is an optional emergency voting phase initiated when equivocation is detected.
* When equivocation occurs, the final observations differ, allowing all honest nodes to identify the equivocation.
* Merge requires certificates from Prepare, making certificate forgery impossible under the signature assumptions.
* Therefore, equivocation must originate from the Prepare phase, where the Byzantine node would need to generate two conflicting certificates.
* The state established during Prepare is shared through Merge, allowing all nodes to identify the equivocation.
* Since both certificates are available to all nodes, they can verify them. If both certificates are valid, the behavior is treated as intentional equivocation.
* Consequently, all \(f+1\) honest nodes vote for expulsion, triggering Kick and causing the node to be removed by all nodes.
* The system then updates \(f'=f-1\) and recalculates the quorum size, total number of nodes, and number of Byzantine nodes.
* This phase formalizes the number of signatures required for Commit.

### Commit

* Each node determines the value to commit based on the final claim calculated during Conflict.
* Abstention means that a node is included in the total vote count but does not vote in favor of any candidate value.
* A node whose signature verification has failed is treated as having abstained.
* The value that receives more than half of the votes is committed. Since all honest nodes produce the same claim and there are at most \(f\) Byzantine nodes, Byzantine nodes cannot obtain a majority and cause conflicting values to be committed.
* Once the final value is determined, each node signs it and propagates the signature to the other nodes.
* A set of \(f+1\) signatures is considered a valid commit certificate. Since all honest nodes produce the same final value, Byzantine nodes cannot obtain \(f+1\) signatures for a conflicting value.
* By assumption, all honest nodes agree on the final value. Therefore, Byzantine nodes cannot obtain \(f+1\) signatures for a conflicting value, and two conflicting commits cannot exist.
* All nodes exchange their signatures with one another, resulting in a message complexity of \(O(n^2)\).
