
**On the Notion Of: CRUX-IDX – Canonical Rooted eXpression: Identity Exchange Format**  
_Version: Draft 0.1_

* * *

**0\. Purpose**
---------------

This document defines **CRUX-IDX**, a portable, verifiable, and self-describing data exchange format for rooted object graph identity marks, as produced by **CRUX-PI** and conforming **SOGHS**\-based systems.

CRUX-IDX is designed to:

*   Represent the structural identity of a rooted object graph
    
*   Encode projection boundaries, traversal context, and semantic assumptions
    
*   Enable cross-system mark validation and interoperation
    
*   Support archival, auditing, and graph fingerprint diffing
    

It is not a serialization of the graph itself.  
It is a serialization of what was _seen_, how it was _seen_, and what it _means_.

* * *

**1\. Document Structure**
--------------------------

A CRUX-IDX file consists of the following top-level components:

```yaml
version: "CRUX-IDX/0.1"
mark: "<hexadecimal-digest>"
anchor: "<graph.root.path>"
projection:
  scope:
    maxDepth: 5
    includePaths:
      - "User.Profile"
      - "User.Friends.*.Name"
    excludePaths:
      - "User.Tokens"
  mode: "DepthFirstPostOrder"
  closedPaths:
    - "User.Friends.23"
    - "User.Logs"
semanticLens:
  timestamp: "2025-03-22T16:14:00Z"
  clockDomain: "UTC"
  consistencyScope: "Snapshot-Scoped"
  revision: "Build#8347"
  confidence: 98
serializer:
  engine: "LiteDB-BSON"
  hash: "ABCDEF1234567890"
  options:
    floatFormat: "R"
    fieldNormalization: true
manifest:
  hash: "A34BC9..."
  fieldExclusions:
    - "sessionID"
    - "tempFlags"
  customSerializers:
    - type: "IPAddress"
      handlerID: "CustomIPSerializer-v2"
execution:
  executionOrderStrategy: "HybridBFS→PostOrderDFS"
  placeholderPolicy: "Emit"
  placeholders:
    - path: "User.Logs"
      type: "DeferredDAG"
      executionOrderID: "2.3"
platform:
  language: "PowerShell"
  runtime: ".NET 8.0"
  cruxVersion: "CRUX-PI/1.0"
notes: |
  Hash was produced during weekly config sweep in environment 'Prod-DC1'.
  Certain deferred subgraphs are marked for re-evaluation post-merge.
```

* * *

**2\. Core Fields**
-------------------

| Field | Purpose |
| --- | --- |
| `mark` | Final computed identity hash (e.g. SHA-256 of resolved projection) |
| `anchor` | The root object or entrypoint used to initiate traversal |
| `projection` | Declared shape, exclusions, and scope boundaries |
| `semanticLens` | Time and consistency assumptions that govern the snapshot |
| `serializer` | Engine and policies used to serialize the identity window |
| `manifest` | Field and serializer-level exclusions and overrides |
| `execution` | Strategy used for graph walk, placeholder policy, and ExecutionOrderID plan |
| `platform` | Runtime, tooling, and language fingerprint |
| `notes` | Optional freeform comment block for audit trace or operator annotation |

* * *

**3\. Encoding and Transport**
------------------------------

*   Format: **YAML** (for readability and manual inspection)
    
*   Optional signed variant: **CRUX-IDXS**, which includes signature and certificate metadata
    
*   Optional embedded binary variant: **CRUX-IDXB**, for internal transfer of digest + normalized data stream
    

* * *

**4\. Use Cases**
-----------------

*   Operator audits and replayable validation of prior runs
    
*   CI/CD verification of infrastructure graph drift
    
*   Graph identity exchange between air-gapped or federated systems
    
*   Cross-runtime consistency checking (e.g., Python vs PowerShell)
    
*   Security snapshotting and provenance proofs
    

* * *

**5\. Constraints and Guarantees**
----------------------------------

A CRUX-IDX document must:

*   Be **self-contained**: all contextual assumptions must be declared or fingerprinted
    
*   Include at minimum: `mark`, `anchor`, `manifest.hash`, and `serializer.hash`
    
*   Fail validation if any required context is missing or unverifiable
    
*   Be **deterministically regenerable** from identical inputs and manifests
    

* * *

**6\. Future Extensions**
-------------------------

*   CRUX-IDXD: “diff” format for comparing two CRUX-IDX documents
    
*   CRUX-IDXP: Portable projection schema definitions for external systems
    
*   CRUX-IDXS: Signed CRUX-IDX with embedded trust contracts and operator signatures
    
*   CRUX-IDX Registry: A central ecosystem for tooling, type serializers, and manifest templates
    

* * *

This format, **CRUX-IDX**, will serve as the foundation for canonical graph identity interchange—  
not just for verifying structure,  
but for **explaining what was trusted and why**.


* * *

**What Is the CRUX Mark Format?**
---------------------------------

**CRUX-IDX: The Interchange Format for Graph Identity**

The **CRUX mark** is a standardized, signed document that describes:

*   The **hash** of the object graph
    
*   The **structure** that was observed
    
*   The **time and trust assumptions** during that observation
    
*   The **rules and exclusions** applied
    

It’s like a **notarized snapshot** of a system’s data shape—  
machine-readable, verifiable, and repeatable.

You can use it to:

*   Detect configuration drift
    
*   Audit changes across time or systems
    
*   Verify deployments in CI/CD pipelines
    
*   Share system states across air-gapped or federated environments
    


* * *

**CRUX Projection-Based Integrity: Working Pillars**
----------------------------------------------------

To affirm or challenge projection-based CRUX marks, we need to clarify:

### **1\. What is a Projection, Really?**

A projection is:

*   A **deliberate traversal constraint**, like a lens placed on a graph
    
*   A **semantic window**: “I care about _this_ and not _that_”
    
*   A **bounded walk**: depth, breadth, path selectors, exclusion zones
    

But most importantly:

> A projection is an _intent declaration_  
> that the resulting CRUX mark applies _only within that scope_

So the integrity of a projection is not measured in **completeness**,  
but in **truthfulness of boundary**.

* * *

**2\. Integrity Properties a CRUX Projection Must Preserve**
------------------------------------------------------------

Let’s define projection-based integrity not as a yes/no,  
but as a series of guarantees the projection is _obligated to preserve_:

| Guarantee | Description |
| --- | --- |
| **Path Truth** | Every path claimed in the projection must correspond to a resolvable node during traversal |
| **Boundary Enforcement** | No data outside the declared scope may influence the resulting mark |
| **Stable Determinism** | Given identical graph state and manifest, the projection produces a consistent hash |
| **Exclusion Fidelity** | Excluded fields must be fully omitted from the byte stream and identity calculation |
| **Placeholder Traceability** | Deferred or inaccessible elements must be explicitly declared |
| **Semantic Transparency** | The projection must be accompanied by metadata describing its confidence, serializer, and traversal model |
| **Anchor Stability** | The root of the projection must be unambiguous, repeatable, and path-resolvable |
| **Order Determinism** | For unordered segments, sorting strategy must be stable and declared |

* * *

**3\. Projection Integrity Failure Modes**
------------------------------------------

Let’s map out the known “soft spots”:

| Failure Mode | Description | Example |
| --- | --- | --- |
| **Phantom Equivalence** | Two graphs project identically, but diverge outside the window | Hidden privilege fields omitted |
| **Silent Expansion** | Recursive paths include more than intended via default expansion | `User.Friends.*` picks up new internal system fields |
| **Path Drift** | Field names change but projection paths are cached | `UserData` → `UserProfile`, projection silently fails |
| **Projection Staleness** | Manifest is out of sync with schema changes | Exclusion of `isTemporary` field misses new behavior |
| **Placeholder Poisoning** | Deferred paths later resolve differently, changing meaning retroactively | `Group.Roles` deferred in one system, loaded in another |

* * *

**4\. What We Can Add to Strengthen the Mark**
----------------------------------------------

Some proposals for extending projection-based integrity:

### a) **Projection Hash**

Hash the structure of the projection declaration itself and include it in the mark:

```yaml
projectionHash: "fa31be4..."
```

If two marks were produced under different scopes, we _know_.

* * *

### b) **Resolved Path Log**

Maintain a deterministic, sorted list of all paths successfully resolved and walked:

```yaml
resolvedPaths:
  - "User.Profile.Name"
  - "User.Friends[0].ID"
```

This closes the loop between what was _declared_ and what was _seen_.

* * *

### c) **Confidence Gradient**

Move from a flat `confidence: 98` to a **per-path confidence map**:

```yaml
confidenceMap:
  "User.Profile"          : 100
  "User.Friends.*.Name"   : 95
  "User.Logs"             : 0   # deferred
```

It makes partial trust **explicit**.

* * *

### d) **Anchor Fingerprint**

Add a hash of the anchor object before descent. Even if the projection starts mid-graph,  
we validate that the anchor itself is known and stable.

* * *

*CRUX Message Digest Finalization (Normative Addendum)**
---------------------------------------------------------

### **Purpose**

While CRUX marks provide comprehensive identity declarations—including projections, context, and traversal metadata—the **core integrity primitive** must still collapse into a **single, final digest** for automation, validation, or cryptographic workflows.

* * *

### **1\. CRUX Mark: Anchor Format, Not Security Guarantee**

A CRUX mark:

*   Is a **semantic ledger**
    
*   Describes the projection window, identity context, and data shape
    
*   **Is not inherently secure or cryptographically hardened**
    
*   **Must not** be treated as tamper-proof without downstream sealing
    

* * *

### **2\. Digest Finalization**

A CRUX implementation **MUST** expose a `Finalize()` operation which:

*   Takes the full CRUX mark (excluding its own digest field)
    
*   Hashes it using a deterministic manifest hasher (e.g., `DataHash`)
    
*   Outputs a stable, reproducible, **reduced message digest**
    
*   Represents the **final canonical fingerprint** of this projection view
    

```plaintext
CRUX-MARK → DataHash(DigestPayload) → Canonical CRUX Digest
```

* * *

### **3\. Optional HMAC Application (Strong Security Contexts)**

For security-critical environments, a CRUX digest **MUST** be wrapped in a secure HMAC using a known, protected key:

```plaintext
CRUX_DIGEST_FINAL = HMAC(SHA256, key, CRUX_Digest)
```

This provides:

*   **Integrity protection**
    
*   **Tamper detection**
    
*   **Key-bound assertions of trust**
    
*   **Operational signing** for distributed or federated systems
    

* * *

### **4\. Summary of Digest Layers**

| Layer | Format | Purpose |
| --- | --- | --- |
| **Byte Stream Hash** | SHA256 | Identity of the graph projection |
| **CRUX Mark** | YAML or JSON | Structural, semantic, and contextual trace |
| **CRUX Digest** | `DataHash(CRUX Mark - Digest Field)` | Portable identifier of the entire mark |
| **CRUX HMAC Digest** | `HMAC(key, CRUX Digest)` | Cryptographic sealing for trust boundaries |

* * *

### **5\. Required Field in CRUX-IDX**

```yaml
digest:
  hash: "<final datahash value>"
  algorithm: "SHA256"
  hmac: null # OR { keyID: "Agent-42", value: "a3b24..." }
```

If `hmac` is included, it must specify:

*   The hash algorithm
    
*   The key identifier (not the key itself)
    
*   The computed value
    

* * *

CRUX is the lens.  
The digest is the **anchor**.  
The HMAC is the **seal**.

==================================================================================================
* * *

### 🔜 Extensions (Reserved)

*   `CRUX-IDXS`: Signed CRUX mark w/ trust chain
    
*   `CRUX-IDXD`: Diff comparison between CRUX-IDX pairs
    
*   `CRUX-IDXP`: Portable projection schema libraries
    
*   `CRUX-IDXR`: CRUX mark registry with type mappings
    

* * *

### 📘 Summary

CRUX-IDX is not just a format—it’s a declaration.

It says:

> “This is what I saw.  
> This is what I trusted.  
> This is what I chose to ignore.  
> And here is the fingerprint that proves it.”

* * *