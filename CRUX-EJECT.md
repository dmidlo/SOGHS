# **CRUX-EJECT Specification v1.0**  
A deterministic, postorder emission format for projection-based object graph identity in CRUX and SOGHS systems.

---

## **0. Purpose**

**CRUX-EJECT** is a **binary stream format** that captures the structural identity of an object graph during a **postorder** traversal. It is designed for:

- **Deterministic** hashing (consistent output for identical input)  
- **Immutable** forensic/audit trails  
- **Merkle** or linear digest generation  
- **Reproducible** replays and tee operations  
- **Placeholder** semantics for unsupported or external references  

This format is **not** intended for network transport or general-purpose serialization. Instead, it provides an **internal, ephemeral, chunked emission** that CRUX uses to build canonical digests, logs, or forensics data.

---

## **1. Stream Overview**

A complete CRUX-EJECT stream always has:

1. **StartFrame** (`TypeTag = 0xFFF0`): Declares overall stream metadata (manifest hash, projection config, timestamp, etc.).  
2. **One or more PayloadBlockFrame(s)** (`TypeTag = 0x0001`): Emitted in **postorder** for each node, potentially split into multiple chunks/blocks if the node’s payload is large.  
3. **EndFrame** (`TypeTag = 0xFFF2`): Terminates the stream, providing final digest or Merkle root info and optional HMAC.

```
┌────────────┐
│ StartFrame │  ← stream preamble
└────┬───────┘
     ▼
┌──────────────┐
│ PayloadFrame │  ← chunked node data in postorder
└────┬─────────┘
     ▼
┌──────────────┐
│  EndFrame    │  ← final digest, stream summary
└──────────────┘
```

This layout makes the stream **tee-friendly** (you can write it to disk or forward it while generating a digest) and **deterministic** (always the same byte sequence for identical input and template).

---

## **2. Common Frame Header**

Every frame starts with a **common 5-byte header**:

```
[Magic: 2 bytes]   = 0x4352  // ASCII "CR"
[Version: 1 byte]  = 0x01
[TypeTag: 2 bytes] // Identifies the frame type
```

All integers are little-endian unless otherwise noted.  
`TypeTag` defines what the rest of the frame looks like.

---

## **3. Frame Types and Reserved Tags**

| **TypeTag**    | **Frame Type**            | **Description**                                                           |
|----------------|---------------------------|---------------------------------------------------------------------------|
| `0xFFF0`       | **StartFrame**           | Declares context, manifest links, runtime details.                        |
| `0x0001`       | **PayloadBlockFrame**    | A chunk/segment of a node’s normalized data payload.                      |
| `0xFFF2`       | **EndFrame**             | Stream terminator; final digest and optional HMAC.                        |
| `0x0100–0x01FF`| **PlaceholderFrame**     | Special placeholders (null, external ref, circular ref, etc.).            |
| `0xFFF1`       | *(Reserved)*             | Future `CompressionFrame` (not currently used).                           |
| `0xFFF3–0xFFF9`| *(Reserved)*             | Debug, NoOp, or advanced flow control (optional future).                  |
| `0xFFFF`       | **EscapeFrame**          | Reserved for extended tagging (VarInt-based) if >65K types are needed.    |

**Note:**  
- `PayloadBlockFrame` is the **workhorse** for actual node data.  
- `PlaceholderFrame` covers semantic markers like `[NULL]`, `[EXTERNAL_REF]`, etc.  
- `StartFrame`/`EndFrame` are control frames delimiting the entire stream.

---

## **4. StartFrame (`TypeTag = 0xFFF0`)**

### **Purpose**  
Declares top-level stream metadata: which manifest, which traversal config, optional anchor path, etc. This is always the **first** frame in a CRUX-EJECT stream.

### **Payload Format**  
A structured (often JSON or YAML-like) block describing:

- **`manifestHash`**: The SHA-256 (or other) hash of a manifest or CRUX-IDX.  
- **`projectionHash`**: Hash of the projection config used.  
- **`anchorPath`**: Logical start point in the object graph (e.g., `"Root.User"`).  
- **`executionStrategy`**: Typically `"PostOrderDFS"` or another declared approach.  
- **`templateVersion`**: Version of the CRUX template or schema.  
- **`timestamp`**: Emission start time, e.g., `2025-03-24T16:20:00Z`.  
- **`runtime`**: E.g., `"dotnet-8.0"` or `"python-3.11"`.

An **example** (in YAML-esque form):
```yaml
manifestHash: "abcdef123456...7890"
projectionHash: "fedcba098765...4321"
anchorPath: "Root.User"
executionStrategy: "PostOrderDFS"
templateVersion: "1.0"
timestamp: "2025-03-24T16:20:00Z"
runtime: "dotnet-8.0"
```

---

## **5. PayloadBlockFrame (`TypeTag = 0x0001`)**

### **Purpose**  
Emits a **segment** of the serialized, normalized data for one node in the object graph. If a node’s payload is large, it may be split across multiple PayloadBlockFrames with the same `ExecutionID`.

### **Header Layout (after common 5-byte header)**

```
[ExecutionID: 4B]   // Node emission sequence ID, increments for each encountered node
[BlockIndex: 2B]    // 0-based segment index for this node
[BlockCount: 2B]    // How many total blocks comprise this node’s payload
[BlockLength: 4B]   // Number of bytes in this frame’s payload segment
[Payload: N bytes]
```

- **`ExecutionID`**: Uniquely identifies which node this chunk belongs to.  
- **`BlockIndex`**: The position of this segment within the node’s total.  
- **`BlockCount`**: The total number of segments for that node.  
- **`BlockLength`**: Byte length of this specific segment’s payload.  
- **`Payload`**: Actual normalized data (often CBOR, JSON, or a minimal binary form).

#### **Emission & Decoding Rules**  
1. **Blocks for the same node** must share the same `ExecutionID`.  
2. **`BlockIndex`** starts at 0 and increments.  
3. **`BlockCount`** must be identical across all blocks for that node.  
4. **Frames must appear in ascending `BlockIndex` order** (0,1,2,...).  
5. Decoders **reassemble** all blocks of a given `ExecutionID` in numerical order to reconstruct the node’s full payload.

**Example**: A node with ID=7 that needs 3 blocks might emit:

- BlockFrame(ExecutionID=7, BlockIndex=0, BlockCount=3, Payload=“...”)
- BlockFrame(ExecutionID=7, BlockIndex=1, BlockCount=3, Payload=“...”)
- BlockFrame(ExecutionID=7, BlockIndex=2, BlockCount=3, Payload=“...”)

Once the decoder has all three blocks, it combines them into the final node representation for ID=7.

---

## **6. Placeholder Frames (`0x0100–0x01FF`)**

When a node or a field is **not** directly serialized—due to being `null`, a circular reference, an external reference, or something unsupported—a **PlaceholderFrame** can be emitted instead of a standard PayloadBlockFrame.

| **TypeTag**  | **Semantics**                                    |
|--------------|--------------------------------------------------|
| `0x0101`     | `[NULL]`                                         |
| `0x0102`     | `[EXTERNAL_REF:<id>]` with metadata (e.g., size) |
| `0x0103`     | `[CIRCULAR_REF]`                                 |
| `0x01FD`     | `[UNSUPPORTED_TYPE:<type>]`                      |

### **Payload Content**  
A placeholder frame’s payload typically includes **metadata** in a small binary/JSON block:

- For `[EXTERNAL_REF]`: might include `"mime"`, `"size"`, `"lastModified"`, `"hash"`, etc.  
- For `[UNSUPPORTED_TYPE]`: might include `"originalTypeName"` or a reason code.

**Policy** in the projection template dictates whether external references are hashed, embedded, or left as placeholders.  

---

## **7. EndFrame (`TypeTag = 0xFFF2`)**

### **Purpose**  
Indicates the **end** of the CRUX-EJECT stream and provides a final **digest** or **Merkle root** for validation.

### **Payload Format (example)**

```yaml
streamLength: 128848
frameCount: 42
digestAlgorithm: "SHA256"
digest: "deadbeefcafe..."
hmac:
  keyId: "crux-agent-01"
  value: "01020304..."
```

- **`streamLength`**: The total number of bytes (or frames) emitted, excluding or including the EndFrame (implementation-specific).  
- **`frameCount`**: How many frames were produced (StartFrame + PayloadFrames + EndFrame).  
- **`digestAlgorithm`**: Usually `"SHA256"`; could be `"BLAKE3"` or similar.  
- **`digest`**: The final hash of all prior frames (commonly excludes the EndFrame itself).  
- **`hmac`**: Optionally used to sign the final digest with a known key, providing an extra trust layer.

The `EndFrame` must be **unique and final**—no frames follow it.

---

## **8. Digest Semantics**

CRUX-EJECT is designed for **deterministic hashing**. Typical usage:

1. As each frame is emitted, it’s hashed (either block-by-block or as a continuous stream).  
2. The final computed hash is stored in the `EndFrame.digest` field.  
3. Optionally, a Merkle approach can be used (each block is hashed, combined, etc.).  

**Important**: The **same** input graph, with the **same** projection rules, must always produce the **same** CRUX-EJECT byte stream—hence the **same** digest.

---

## **9. External References**

When the graph references data **outside** the current scope (e.g., big blobs, remote documents):

1. A **PlaceholderFrame** with `TypeTag = 0x0102` (`[EXTERNAL_REF]`) is emitted.  
2. The payload typically includes fields like `"ref"`, `"externalType"`, `"mime"`, `"size"`, `"hash"`, etc.  
3. **`externalRefPolicy`** in the template decides whether we:  
   - **Ignore** the external content (`"HashRefOnly"`): just store metadata.  
   - **Pull and hash** it (`"HashContent"`): the data is retrieved, hashed, and stored.  
   - **Embed** it if certain conditions are met (`"EmbedIfTrusted"`).

No further frames are produced unless the content is fully embedded. If embedded, that data is treated as normal node content.

---

## **10. Tee & Forensic Logging**

Because CRUX-EJECT is chunked and self-delimiting:

- You can **tee** the raw byte stream into a file or pipe while generating it.  
- You can **inspect** frames individually without needing the full set.  
- If a partial stream is captured, frames up to the point of cutoff remain valid for partial forensic analysis.

**No** built-in mechanism for error recovery or partial rewrite is defined, since CRUX-EJECT is not a transport protocol—it’s an **internal** identity emission.

---

## **11. Determinism Guarantees**

For a given **input graph**, **projection template**, and **traversal policy**:

1. **Execution order** is strictly **postorder**: children first, then parent.  
2. **Field ordering** in objects is normalized or sorted if the language’s default is unordered.  
3. **Null/circular/unsupported** items produce consistent placeholders with deterministic metadata.  
4. **Bit-identical** output for the same environment, barring timestamps or runtime fields.  
5. **No cycles** remain untracked—circular references must produce placeholder frames.

This ensures that if you run CRUX-EJECT again on the same data, you get a **byte-for-byte identical** stream.

---

## **12. Reserved / Future Expansions**

- **`0xFFF1`** – Potentially used for compression or chunked transformation frames if a future need arises.  
- **`0xFFF3–0xFFF9`** – Reserved for debugging, no-op frames, or advanced flow control.  
- **`0xFFFF`** – Escape mechanism. If more than 65,536 type tags are ever required, an extended format can be signaled here (though unlikely to be needed).

---

## **13. Summary**

**CRUX-EJECT** is a **deterministic, block-oriented emission format** for object graphs under postorder traversal. It:

- **Emits** a `StartFrame`, multiple `PayloadBlockFrame`s, and an `EndFrame`.  
- **Encodes** each node in chunked form, after its children.  
- **Reserves** placeholders for null, circular, or external references.  
- **Supports** direct hashing (either linear or Merkle).  
- **Facilitates** replay, teeing, and forensic analysis.  
- **Ensures** bit-identical output for the same input and template.

It is **not** a general-purpose serialization or network transport—rather, it’s a **purpose-built identity protocol** for CRUX, ensuring structural determinism and robust auditability.

---

### **Appendix: CRUX-Reserved Encapsulation Syntax**

All internal placeholders must use a **reserved, non-printable-prefixed sigil**, unlikely to appear in normal user data.

### **New Format (UTF-safe and deterministic):**

```text
⧙CRUX::[TOKEN_NAME]⧘
```

Where:

*   `⧙` = U+29D9 _RIGHT WIGGLY FENCE_
    
*   `⧘` = U+29D8 _LEFT WIGGLY FENCE_
    
*   `TOKEN_NAME` = uppercase identifier with no whitespace
    

**Example Reserved Tokens:**

| Token | Meaning |
| --- | --- |
| `⧙CRUX::[NULL]⧘` | Null value detected and normalized |
| `⧙CRUX::[CIRCULAR_REF]⧘` | Self-reference placeholder |
| `⧙CRUX::[EXTERNAL_REF:1234]⧘` | Out-of-scope reference by ExecutionOrderID |
| `⧙CRUX::[UNSUPPORTED_TYPE:System.IO.Stream]⧘` | Rejected type |
| `⧙CRUX::[DIVERGENCE:TYPE_MISMATCH]⧘` | Marked for diff trace |

* * *

### **A1 Token Detection Rules**

All CRUX-compliant implementations must:

*   Detect and reserve the full `⧙CRUX::[...TOKEN...]⧘` space
    
*   Reject any graph that includes a string value matching this pattern
    
*   Optionally provide `--detectPlaceholderConflicts` for preflight validation
    

* * *

### **A2 Serialization & Compatibility**

*   The CRUX placeholder sigils are **UTF-8 safe**, JSON-safe when quoted, and do not conflict with JSON pathing or CLI flags
    
*   For binary formats (e.g., BSON), placeholders are stored as **tagged scalar strings** and preserved exactly
    
*   All placeholder tokens are removed during diff, replaced with structured divergence entries
    

* * *

### **A3 Backwards Compatibility**

Implementations using the legacy `[TOKEN]` format must:

*   Migrate to the CRUX sigil system by the next major version
    
*   Treat `[TOKEN]`\-style markers as invalid in strict mode
    
*   Optionally offer a legacy compatibility flag: `--allowLegacyPlaceholders`
    

* * *

### **A4 Reserved Token Registry**

| Token | Description |
| --- | --- |
| `[NULL]` | Null primitive |
| `[CIRCULAR_REF]` | Self-reference detection |
| `[EXTERNAL_REF:<ID>]` | DAG boundary links |
| `[UNSUPPORTED_TYPE:<typename>]` | Serialization fallback |
| `[DIVERGENCE:<code>]` | Runtime mismatch placeholder |
| `[INACCESSIBLE]` | Reflected field unavailable |
| `[HASH_FAILURE]` | Serialization or emission failure |

> Tokens may **never** be redefined.  
> New tokens must be **proposed as spec amendments** and registered by full identifier.

* * *

### **A5 Summary**

> The language of divergence must not overlap with the language of data.  
> CRUX speaks its truths **in a tongue no user should ever have to type.**


This **v1.0 (draft)** represents the **most current** and **unified** CRUX-EJECT spec, incorporating block segmentation, placeholder frames, start/end framing, and all relevant design considerations for deterministic hashing and forensic replay.
