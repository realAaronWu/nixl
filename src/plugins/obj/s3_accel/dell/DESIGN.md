<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Dell ObjectScale S3-over-RDMA Engine -- Refactoring Design

This document describes a refactoring of the Dell ObjectScale accelerated
engine from cuObject **Pattern A** (callback-based I/O) to **Pattern B**
(manual RDMA token management).  The refactoring reduces Dell-specific code
by roughly 70%, eliminates API misuse, and integrates cleanly with the
existing `DefaultObjEngineImpl` transfer pipeline.

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Background: cuObject Patterns A and B](#2-background-cuobject-patterns-a-and-b)
3. [Current Architecture (Pattern A)](#3-current-architecture-pattern-a)
4. [Proposed Architecture (Pattern B)](#4-proposed-architecture-pattern-b)
5. [Key Design Decisions](#5-key-design-decisions)
   - [5.1. Pattern B over Pattern A](#51-pattern-b-over-pattern-a)
   - [5.2. Override putObjectAsync/getObjectAsync, not separate RDMA methods](#52-override-putobjectasyncgetobjectasync-not-separate-rdma-methods)
   - [5.3. Inherit prepXfer/postXfer/checkXfer/releaseReqH from the parent](#53-inherit-prepxferpostxfercheckxferreleasereqh-from-the-parent)
   - [5.4. Separate CuObjTokenManager class](#54-separate-cuobjtokenmanager-class)
   - [5.5. DRAM/VRAM metadata in the Dell engine, OBJ_SEG metadata in the parent](#55-dramvram-metadata-in-the-dell-engine-obj_seg-metadata-in-the-parent)
   - [5.6. RDMA token generated inside putObjectAsync/getObjectAsync](#56-rdma-token-generated-inside-putobjectasyncgetobjectasync)
   - [5.7. Per-request fallback to HTTP body transfer](#57-per-request-fallback-to-http-body-transfer)
   - [5.8. Zero-copy data plane](#58-zero-copy-data-plane)
   - [5.9. Pool-level registration over per-buffer registration](#59-pool-level-registration-over-per-buffer-registration)
6. [Detailed Component Design](#6-detailed-component-design)
7. [End-to-End Transfer Flow](#7-end-to-end-transfer-flow)
8. [File Change Summary](#8-file-change-summary)
9. [Migration Checklist](#9-migration-checklist)

---

## 1. Motivation

The current Dell engine (`S3DellObsObjEngineImpl`) works, but carries
unnecessary complexity and technical debt.

**Problem 1 -- API misuse.**  The engine uses Pattern A's `cuObjPut()` /
`cuObjGet()` synchronous I/O calls purely to trigger callbacks that extract
RDMA descriptors.  The callbacks (`objectGet`, `objectPut`) do not perform
any server communication -- they return `0` immediately after copying the
descriptor string from `cufileRDMAInfo_t`.  This contradicts the cuObject
spec (sections 1.6, 1.10), which states that callbacks "must communicate
with cuObjServer using the RDMA descriptor information" and return the
number of bytes transferred.  The library interprets a return of `0` as
"zero bytes transferred", which happens to not fail but is semantically
incorrect.

**Problem 2 -- unnecessary lifecycle reimplementation.**  The engine
overrides six virtual methods (`registerMem`, `deregisterMem`, `prepXfer`,
`postXfer`, `checkXfer`, `releaseReqH`) with 600+ lines of code.  Four of
these (`prepXfer`, `postXfer`, `checkXfer`, `releaseReqH`) duplicate logic
that already exists in `DefaultObjEngineImpl`.  The duplication exists
solely because the current design separates RDMA descriptor extraction
(in `prepXfer`) from the S3 request (in `postXfer`), requiring custom
request handle classes to carry descriptors across the two phases.

**Problem 3 -- runtime dynamic_cast.**  `postXfer` must
`dynamic_cast<iDellS3RdmaClient*>(s3Client_.get())` on every transfer to
access Dell-specific RDMA methods (`putObjectRdmaAsync`,
`getObjectRdmaAsync`).  This is fragile and fails silently if the cast
returns nullptr (currently logged as an error but still returns
`NIXL_IN_PROG`, not `NIXL_ERR_BACKEND`).

**Problem 4 -- descriptor lifetime concern.**  The cuObject spec (section
1.15) states: "RDMA descriptors are only valid during the callback
invocation."  The current code copies `infop->desc_str` inside the
callback and uses the *copy* later.  This works today because the spec
refers to the pointer, not the content -- but it relies on an assumption
about library internals that the spec does not guarantee.

**Problem 5 -- no per-request fallback.**  The current engine is
all-or-nothing: it is selected at init time by the factory in
`obj_backend.cpp`, and if any RDMA operation fails (`cuObjPut` /
`cuObjGet` in `prepXfer`, or the S3 async call in `postXfer`), the
transfer returns `NIXL_ERR_BACKEND` with no retry path.  Transient RDMA
failures -- NIC resource exhaustion, firmware resets, cable events --
cause hard errors rather than graceful degradation.  By contrast,
LMCache's `CuObjectS3Connector` falls back to standard HTTP body
transfer **per-request**, so a single RDMA token failure only affects that
one request while all others continue using RDMA.

---

## 2. Background: cuObject Patterns A and B

The cuObjClient API (v1.0.0, CUDA Toolkit >= 13.1.1) offers two patterns
for S3-over-RDMA:

### Pattern A -- Callback-based I/O (spec sections 1.2, 1.6, 1.10)

`cuObjPut()` and `cuObjGet()` are **synchronous** I/O operations.  The
library invokes user-supplied `CUObjIOOps` callbacks, passing a
`cufileRDMAInfo_t*` containing the RDMA descriptor.  The callback is
expected to perform the **entire server round-trip** synchronously -- send
the descriptor, wait for the RDMA data transfer, and return the byte count.
The library blocks until the callback returns.

### Pattern B -- Manual RDMA Token Management (spec section 1.12.4)

`cuMemObjGetRDMAToken()` generates an RDMA token string for a sub-region
of a registered buffer.  The caller owns the token lifetime -- it copies
the string and frees the library allocation via `cuMemObjPutRDMAToken()`.
No callbacks.  No synchronous I/O.  The caller injects the token into its
own HTTP request flow.

```cpp
// Pattern B usage (from cuObject spec section 1.12.4):
char* rdma_token = nullptr;
client.cuMemObjGetRDMAToken(buffer, region_size, region_offset,
                            CUOBJ_GET, &rdma_token);
// ... use rdma_token in HTTP header ...
client.cuMemObjPutRDMAToken(rdma_token);
```

**This refactoring adopts Pattern B.**

---

## 3. Current Architecture (Pattern A)

### Class Hierarchy

```
nixlObjEngineImpl (abstract)
  +-- DefaultObjEngineImpl
        +-- S3AccelObjEngineImpl
              +-- S3DellObsObjEngineImpl          <-- overrides 6 virtual methods

iS3Client (abstract)
  +-- awsS3Client
        +-- awsS3AccelClient
              +-- awsS3DellObsClient              <-- also implements iDellS3RdmaClient
```

### Dell-Specific Artifacts (to be eliminated or simplified)

| Artifact                      | Purpose in Pattern A                                           |
|-------------------------------|----------------------------------------------------------------|
| `rdma_interface.h`            | `iDellS3RdmaClient` abstract interface with `putObjectRdmaAsync` / `getObjectRdmaAsync` |
| `rdma_ctx_t`                  | Struct to capture RDMA descriptor inside callback              |
| `objectGet` / `objectPut`     | Static `CUObjIOOps` callbacks that extract `infop->desc_str`   |
| `obs_ops`                     | Global `CUObjIOOps` instance wired to the callbacks            |
| `obsObjTransferRequestH`      | Per-descriptor struct carrying addr, size, offset, rdma_desc, obj_key |
| `nixlObsObjBackendReqH`       | Custom request handle with `vector<obsObjTransferRequestH>` + futures |
| `nixlObsObjMetadata`          | Custom metadata for DRAM/VRAM with localAddr field             |
| `isValidPrepXferParams`       | Duplicate of the same function in `DefaultObjEngineImpl`       |
| `prepXfer` override           | Calls `cuObjPut`/`cuObjGet` to trigger callbacks, builds request handle |
| `postXfer` override           | `dynamic_cast` to `iDellS3RdmaClient`, dispatches RDMA methods |
| `checkXfer` override          | Identical to `DefaultObjEngineImpl::checkXfer`                 |
| `releaseReqH` override        | Identical to `DefaultObjEngineImpl::releaseReqH`               |

---

## 4. Proposed Architecture (Pattern B)

### Class Hierarchy

```
nixlObjEngineImpl (abstract)                           <-- NO CHANGES
  +-- DefaultObjEngineImpl                             <-- NO CHANGES
        +-- S3AccelObjEngineImpl                       <-- NO CHANGES
              +-- S3DellObsObjEngineImpl               <-- overrides only 3 methods

iS3Client (abstract)                                   <-- NO CHANGES
  +-- awsS3Client
        +-- awsS3AccelClient
              +-- awsS3DellObsClient                   <-- overrides putObjectAsync / getObjectAsync
```

### New Component

```
CuObjTokenManager                                     <-- RAII wrapper for Pattern B
  +-- registerMemory / deregisterMemory
  +-- generatePutToken / generateGetToken
```

### Core Idea

RDMA is a **data-plane** concern.  It changes *how* the S3 HTTP request is
constructed (inject an `x-rdma-info` header, set `ContentLength(0)` for
PUT), not *when* or *in what order* transfer lifecycle methods are called.

Therefore:

- The **engine** (`S3DellObsObjEngineImpl`) only overrides `registerMem`,
  `deregisterMem`, `getSupportedMems`, and `getClient`.  It adds cuObject
  memory registration for DRAM/VRAM segments on top of the parent's OBJ_SEG
  key mapping.

- The **client** (`awsS3DellObsClient`) overrides the standard `iS3Client`
  methods `putObjectAsync` and `getObjectAsync` to generate RDMA tokens
  and inject them into the S3 request.  No separate RDMA interface.

- `prepXfer`, `postXfer`, `checkXfer`, and `releaseReqH` are **inherited
  unchanged** from `DefaultObjEngineImpl`.  The parent's `postXfer` already
  iterates descriptors and calls `getClient()->putObjectAsync(...)` with a
  future/promise bridge -- exactly what we need.

---

## 5. Key Design Decisions

### 5.1. Pattern B over Pattern A

**Decision:** Use `cuMemObjGetRDMAToken()` for direct token generation
instead of `cuObjPut()`/`cuObjGet()` callbacks.

**Rationale:**

1. **Correct API usage.**  Pattern A callbacks are documented to perform the
   entire server round-trip synchronously (spec sections 1.6, 1.10).  Using
   them as descriptor-extraction hooks contradicts their contract.  Pattern
   B's `cuMemObjGetRDMAToken()` (spec section 1.12.4) is explicitly designed
   for callers who manage their own HTTP request flow -- which is exactly
   what NIXL does.

2. **Eliminates the prep/post split.**  Pattern A forces RDMA setup into
   `prepXfer` (to invoke the synchronous callbacks) and actual S3 requests
   into `postXfer`, requiring custom request handle classes to carry
   descriptors between the two phases.  Pattern B generates the token at
   the point of use (inside `putObjectAsync`/`getObjectAsync`), so the
   entire transfer is self-contained within `postXfer`.

3. **Explicit token lifetime.**  Pattern B tokens are freed via
   `cuMemObjPutRDMAToken()` at a time the caller controls.  Pattern A
   descriptors are "only valid during the callback invocation" (spec
   section 1.15) -- the current code copies the string, which works but
   relies on implementation assumptions.

4. **Thread safety by design.**  `cuMemObjGetRDMAToken()` writes to a
   caller-provided `char**` output pointer (spec section 1.5.4).  Each
   concurrent caller gets its own allocation with no shared state.
   Pattern A callbacks share context via `cuObjClient::getCtx(handle)`,
   which is safe per-request but requires careful context lifecycle
   management.

### 5.2. Override putObjectAsync/getObjectAsync, not separate RDMA methods

**Decision:** `awsS3DellObsClient` overrides the standard `iS3Client`
virtual methods `putObjectAsync` and `getObjectAsync` instead of
introducing a separate `iDellS3RdmaClient` interface.

**Rationale:**

1. **Eliminates dynamic_cast.**  `DefaultObjEngineImpl::postXfer` already
   calls `getClient()->putObjectAsync(...)`.  If the Dell client overrides
   these methods to inject RDMA tokens, the parent's `postXfer` works
   without modification.  No `dynamic_cast`, no separate dispatch path.

2. **Preserves the Liskov Substitution Principle.**  The Dell client *is-a*
   `iS3Client`.  It fulfills the same contract: take a key, pointer, size,
   offset, and callback; deliver the data asynchronously.  The fact that it
   uses RDMA internally is an implementation detail invisible to callers.

3. **Simplifies testing.**  Mock clients only need to implement the standard
   `iS3Client` interface.  No separate `mockDellS3Client` with RDMA-specific
   methods is needed.

### 5.3. Inherit prepXfer/postXfer/checkXfer/releaseReqH from the parent

**Decision:** Do not override `prepXfer`, `postXfer`, `checkXfer`, or
`releaseReqH` in the Dell engine.

**Rationale:**

1. **These methods are not Dell-specific.**  The parent's `prepXfer` creates
   an empty request handle and validates parameters.  The parent's `postXfer`
   iterates descriptors, looks up object keys, calls the client's async
   methods, and bridges callbacks to futures.  The parent's `checkXfer` polls
   futures.  The parent's `releaseReqH` deletes the handle.  None of this
   logic changes for Dell.

2. **Single source of truth.**  If the parent's transfer pipeline is
   improved (e.g., better error handling, batching optimizations), the Dell
   engine automatically benefits.

3. **~480 fewer lines of Dell-specific code.**  The current `prepXfer` (80
   lines), `postXfer` (60 lines), `checkXfer` (10 lines), and `releaseReqH`
   (10 lines) overrides are eliminated, along with the `obsObjTransferRequestH`
   and `nixlObsObjBackendReqH` classes they require.

### 5.4. Separate CuObjTokenManager class

**Decision:** Extract cuObject lifecycle management (construction, memory
registration, token generation) into a dedicated `CuObjTokenManager` class,
rather than embedding it directly in the engine or client.

**Rationale:**

1. **Single responsibility.**  The engine manages NIXL lifecycle (register/
   deregister memory segments, supported memory types).  The client manages
   S3 requests (PutObject, GetObject).  The token manager manages cuObject
   state (RDMA connection, memory descriptors, token generation).  Each class
   has one reason to change.

2. **Shared ownership.**  Both the engine (for `registerMemory` /
   `deregisterMemory`) and the client (for `generatePutToken` /
   `generateGetToken`) need access to the same `cuObjClient` instance.  A
   shared `CuObjTokenManager` makes this explicit via `shared_ptr`.

3. **Testability.**  The token manager can be mocked independently for unit
   tests without requiring a real cuObject library.

### 5.5. DRAM/VRAM metadata in the Dell engine, OBJ_SEG metadata in the parent

**Decision:** For `OBJ_SEG` registration, delegate to the parent class
(`S3AccelObjEngineImpl::registerMem`).  For `DRAM_SEG` / `VRAM_SEG`,
create Dell-specific metadata that tracks the registered address for later
deregistration.

**Rationale:**

1. **Reuse existing OBJ_SEG handling.**  The parent's `registerMem` already
   manages the `devIdToObjKey_` mapping for `OBJ_SEG` segments.  Duplicating
   this logic is unnecessary and error-prone.

2. **DRAM/VRAM registration is Dell-specific.**  The base `DefaultObjEngineImpl`
   sets `out = nullptr` for DRAM segments because standard S3 transfers do
   not need memory registration.  Dell RDMA *does* need `cuMemObjGetDescriptor`
   to register the buffer, so the Dell engine returns a metadata object that
   stores the address for later `cuMemObjPutDescriptor` cleanup.

3. **Clean deregistration dispatch.**  In `deregisterMem`, the Dell engine
   checks if the metadata is its own (DRAM/VRAM with cuObject registration)
   or the parent's (OBJ_SEG with key mapping).  It handles its own metadata
   and delegates the rest to the parent.

### 5.6. RDMA token generated inside putObjectAsync/getObjectAsync

**Decision:** Generate the RDMA token at the point of use, inside the
client's async methods, not during `prepXfer`.

**Rationale:**

1. **Minimal token lifetime.**  The token encodes RDMA memory addresses and
   NIC connection info.  Generating it immediately before the S3 request and
   not storing it in a request handle minimises the window during which stale
   tokens could be used.

2. **No cross-phase state.**  The current design generates descriptors in
   `prepXfer` and uses them in `postXfer`, requiring custom request handle
   classes to carry them across.  Generating in-place eliminates this
   coupling.

3. **Self-contained error handling.**  If token generation fails, the
   client's async method can fall back to HTTP (see 5.7) or call
   `callback(false)` immediately.  The parent's future/promise bridge
   translates failures to `NIXL_ERR_BACKEND`.  No special error path is
   needed in the engine.

### 5.7. Per-request fallback to HTTP body transfer

**Decision:** When RDMA token generation fails for a single request,
`awsS3DellObsClient` falls back to the parent class's standard HTTP body
transfer (`awsS3AccelClient::putObjectAsync` / `getObjectAsync`) for that
request.  All other concurrent requests continue using RDMA.

**Rationale:**

1. **Transient RDMA failures should not kill the transfer.**  RDMA resource
   exhaustion, NIC firmware resets, and cable flaps are transient events.
   A per-request fallback means the system degrades gracefully: the
   affected request completes via HTTP (slower but correct), while every
   other request in the batch still uses RDMA.  The current Pattern A
   implementation returns `NIXL_ERR_BACKEND` for the entire transfer on
   any RDMA failure.

2. **The inheritance chain makes it trivial.**  `awsS3DellObsClient`
   inherits from `awsS3AccelClient`, which inherits from `awsS3Client`.
   The parent's `putObjectAsync` / `getObjectAsync` methods perform
   standard HTTP body transfers.  Calling `awsS3AccelClient::putObjectAsync`
   (or the `awsS3Client` grandparent) inside a catch block is a single
   line of code -- no new abstractions, interfaces, or flags required.

3. **Matches the LMCache pattern that is proven in production.**
   LMCache's `CuObjectS3Connector._s3_upload` / `._s3_download` use the
   same structure:
   ```python
   def _s3_upload(self, key_str, memory_obj):
       if not self._rdma_enabled:
           return super()._s3_upload(key_str, memory_obj)
       try:
           return self._rdma_upload(key_str, memory_obj)
       except Exception:
           return super()._s3_upload(key_str, memory_obj)
   ```
   The C++ equivalent in `awsS3DellObsClient` is:
   ```cpp
   void awsS3DellObsClient::putObjectAsync(..., callback) {
       try {
           std::string token = tokenMgr_->generatePutToken(ptr, len, 0);
           // ... build RDMA request, send ...
       } catch (const std::exception& e) {
           NIXL_WARN << "RDMA token failed, falling back to HTTP: " << e.what();
           awsS3AccelClient::putObjectAsync(key, data_ptr, data_len, offset, callback);
       }
   }
   ```

4. **Invisible to the engine and the NIXL pipeline.**  The fallback is
   entirely inside the client.  `DefaultObjEngineImpl::postXfer` calls
   `getClient()->putObjectAsync(...)` and gets a callback with
   `success=true` or `success=false` -- it does not know or care whether
   RDMA or HTTP was used.  No changes to the engine, the request handle,
   or the future/promise bridge.

5. **Observable via logging.**  Each fallback emits a `NIXL_WARN` log so
   operators can detect RDMA degradation.  A future enhancement could add
   a counter metric for monitoring dashboards.

**What this does NOT do:**

- It does not add an init-time fallback.  If `CuObjTokenManager` fails
  to connect at construction time (no RDMA hardware, library missing),
  the engine still fails to initialise.  The factory in `obj_backend.cpp`
  should handle this case by falling back to `S3AccelObjEngineImpl` or
  `DefaultObjEngineImpl` if the Dell engine constructor throws.  This is
  a separate concern and can be addressed independently.

- It does not retry the RDMA path.  Each failed request falls back to
  HTTP exactly once.  Subsequent requests still attempt RDMA first.  This
  avoids a "retry storm" under sustained RDMA failure while allowing
  fast recovery when the failure is truly transient.

- **VRAM fallback is not safe.**  The HTTP fallback path calls
  `awsS3AccelClient::putObjectAsync`, which wraps `data_ptr` in an
  `Aws::Utils::Stream::PreallocatedStreamBuf`.  If `data_ptr` points to
  GPU device memory (VRAM_SEG), the AWS SDK will attempt to read/write
  the pointer from CPU context, causing a segfault or silent corruption.
  The fallback must therefore be **gated on memory type**: DRAM_SEG
  requests can fall back to HTTP; VRAM_SEG requests must fail immediately
  (`callback(false)`) with a clear error log.  See section 5.8.

### 5.8. Zero-copy data plane

**Decision:** The RDMA data path must be fully zero-copy for both DRAM
and VRAM.  The HTTP fallback path is zero-copy for DRAM (via
`PreallocatedStreamBuf`) but unavailable for VRAM.

**Analysis of every allocation and copy in the data plane:**

#### RDMA path (PUT)

| Step | What happens | Data copy? |
|------|-------------|------------|
| `cuMemObjGetRDMAToken(ptr, size, offset, CUOBJ_PUT, &desc)` | Library generates RDMA token string | No (control-plane metadata, ~100--500 bytes) |
| `std::string token(desc_str)` | Copy token string into `std::string` | No data copy (token is metadata) |
| `cuMemObjPutRDMAToken(desc_str)` | Free library-allocated string | N/A |
| `PutObjectRequest` + `SetContentLength(0)` | HTTP request with empty body | **No data in HTTP body** |
| Server issues `RDMA_READ` from `data_ptr` | NIC DMA reads directly from registered memory | **Zero-copy** |

**Total data copies: 0.**  The user's buffer at `data_ptr` (DRAM or VRAM)
is read directly by the RDMA NIC via DMA.  No `memcpy`, no kernel buffer,
no HTTP body.

#### RDMA path (GET)

| Step | What happens | Data copy? |
|------|-------------|------------|
| `cuMemObjGetRDMAToken(ptr, size, offset, CUOBJ_GET, &desc)` | Library generates RDMA token | No data copy |
| `GetObjectRequest` + `x-rdma-info` header | HTTP request with no `on_body` handler | **No data in HTTP body** |
| Server issues `RDMA_WRITE` into `data_ptr` | NIC DMA writes directly into registered memory | **Zero-copy** |

**Total data copies: 0.**  Data lands directly in the user's buffer.

#### HTTP fallback path (DRAM only)

The parent class `awsS3Client::putObjectAsync` wraps `data_ptr` in:
```cpp
auto buf = Aws::MakeShared<Aws::Utils::Stream::PreallocatedStreamBuf>(
    "PutObjectStreamBuf", reinterpret_cast<unsigned char*>(data_ptr), data_len);
auto stream = Aws::MakeShared<Aws::IOStream>("PutObjectInputStream", buf.get());
request.SetBody(stream);
```

`PreallocatedStreamBuf` does **not** copy the data.  It wraps the caller's
buffer as the stream's backing store.  The AWS SDK reads directly from
`data_ptr` when serialising the HTTP body.

Similarly, `awsS3Client::getObjectAsync` uses `PreallocatedStreamBuf` as
the response stream factory -- the SDK writes HTTP body bytes directly
into `data_ptr`.

**Total data copies in fallback: 0 at the application level.**  The only
copies are inside the kernel TCP/IP stack (socket send/receive buffers),
which are inherent to non-RDMA HTTP transfer.

#### VRAM and the fallback boundary

`PreallocatedStreamBuf` interprets `data_ptr` as a CPU-addressable
`unsigned char*`.  If `data_ptr` is CUDA device memory (VRAM_SEG),
any CPU-side access will segfault.

**Therefore the per-request fallback in `awsS3DellObsClient` must check
the memory type before falling back:**

```cpp
void awsS3DellObsClient::putObjectAsync(
        std::string_view key, uintptr_t data_ptr, size_t data_len,
        size_t offset, put_object_callback_t callback) {
    if (data_len == 0 || offset != 0) { callback(false); return; }

    try {
        std::string token = tokenMgr_->generatePutToken(
            reinterpret_cast<void*>(data_ptr), data_len, 0);
        // ... RDMA request ...
    } catch (const std::exception& e) {
        // Fallback is only safe for CPU-addressable memory
        auto mem_type = cuObjClient::getMemoryType(
            reinterpret_cast<const void*>(data_ptr));
        if (mem_type == CUOBJ_MEMORY_CUDA_DEVICE) {
            NIXL_ERROR << "RDMA put failed for VRAM, no HTTP fallback: "
                       << e.what();
            callback(false);
            return;
        }
        NIXL_WARN << "RDMA put failed, falling back to HTTP: " << e.what();
        awsS3AccelClient::putObjectAsync(key, data_ptr, data_len,
                                          offset, callback);
    }
}
```

This ensures:
- **DRAM_SEG**: RDMA fails -> HTTP fallback (zero-copy via
  `PreallocatedStreamBuf`) -> request succeeds.
- **VRAM_SEG**: RDMA fails -> `callback(false)` -> `NIXL_ERR_BACKEND`.
  No unsafe CPU access to GPU memory.

The `cuObjClient::getMemoryType()` static method (spec section 1.8.2) is
a lightweight pointer classification check (it queries the CUDA driver
without allocating or copying).

### 5.9. Pool-level RDMA registration despite per-page NIXL descriptors

**Decision:** The `CuObjTokenManager` de-duplicates `cuMemObjGetDescriptor`
calls so that many per-page `registerMem` invocations from NIXL's core
result in a single cuObject RDMA registration for the entire contiguous
buffer.  Token generation computes pool-relative offsets automatically.

#### Why NIXL registers per-page, not per-pool

NIXL's transfer model requires per-page descriptors.  This is not a
registration limit — it is architecturally necessary for how transfers are
composed.  The call chain is:

```
LMCache: NixlStorageAgent.init_mem_handlers()
  → splits buffer into page-size chunks (align_bytes = KV cache chunk size)
  → nixl_agent.register_memory(reg_list, mem_type="DRAM")
      → nixlAgent::registerMem(descs)           [core/nixl_agent.cpp]
          → nixlLocalSection::addDescList()      [infra/nixl_memory_section.cpp:133]
              → for each descriptor i:
                  backend->registerMem(mem_elms[i], nixl_mem, metadata)
```

The per-page split exists because `make_prepped_xfer(op, mem_handler,
mem_indices, storage_handler, storage_indices)` pairs **individual page
indices** from the memory handler with **individual object indices** from
the storage handler.  Each KV cache chunk is transferred independently to
its own S3 object.  A single whole-buffer registration would create only
one transfer descriptor, making it impossible to address individual pages
within it.

The comment in LMCache confirms this:
```python
# Break the registration into page size chunks to ensure the maximum
# buffer size of the underlying plugin is not exceeded
```

So the per-page `registerMem` calls are a **fixed constraint** of the
NIXL-LMCache integration.  We cannot change LMCache to register the pool
once without redesigning NIXL's transfer descriptor model.

#### The problem this creates for cuObject

Each `registerMem(DRAM_SEG)` call in the current Dell engine calls
`cuMemObjGetDescriptor(page_addr, page_size)`.  This is a heavyweight NIC
operation: it pins memory, creates an RDMA Memory Region (MR), and sets
up protection keys.

Typical numbers:
- `buffer_size`: 1--4 GB (configurable via `nixl_buffer_size`)
- `page_size` (`align_bytes`): 256 KB -- 4 MB (model-dependent KV chunk)
- **Number of pages**: 256 -- 16,384
- **Cost per `cuMemObjGetDescriptor`**: ~0.1 -- 1 ms (NIC MR creation)
- **Total registration time**: 0.025 -- 16 seconds

By contrast, LMCache's connector pipeline calls `cuMemObjGetDescriptor`
**once** for the entire pool:
```python
base_ptr, size_bytes = _get_allocator_buffer_info(allocator)
self._cuobj_client.register_pool(base_ptr, size_bytes)  # ONE call
```

Per-transfer, it only calls `cuMemObjGetRDMAToken(pool_base, size,
offset)` which is lightweight (no NIC MR creation, just token string
generation).

#### Solution: transparent de-duplication in CuObjTokenManager

Since we cannot change NIXL's per-descriptor `registerMem` calls, the
`CuObjTokenManager` absorbs the redundancy transparently:

```cpp
cuObjErr_t CuObjTokenManager::registerMemory(void* ptr, size_t size) {
    std::lock_guard<std::mutex> lock(regions_mutex_);

    uintptr_t addr = reinterpret_cast<uintptr_t>(ptr);
    uintptr_t end  = addr + size;

    // Check if [addr, end) is within an already-registered region
    for (auto& region : regions_) {
        if (addr >= region.base && end <= region.base + region.size) {
            region.refcount++;           // Already covered — no cuObject call
            return CU_OBJ_SUCCESS;
        }
    }

    // Check if this new region extends or overlaps an existing one —
    // if so, merge into a single larger registration
    for (auto& region : regions_) {
        if (addr >= region.base && addr <= region.base + region.size) {
            // Contiguous or overlapping extension
            size_t new_end = std::max(region.base + region.size, end);
            size_t new_size = new_end - region.base;
            if (new_size < CUOBJ_MAX_MEMORY_REG_SIZE) {
                // Deregister old, re-register merged
                client_->cuMemObjPutDescriptor(reinterpret_cast<void*>(region.base));
                cuObjErr_t rc = client_->cuMemObjGetDescriptor(
                    reinterpret_cast<void*>(region.base), new_size);
                if (rc == CU_OBJ_SUCCESS) {
                    region.size = new_size;
                    region.refcount++;
                }
                return rc;
            }
        }
    }

    // Truly new region — register with cuObject
    if (size >= CUOBJ_MAX_MEMORY_REG_SIZE) {
        return CU_OBJ_FAIL;
    }
    cuObjErr_t rc = client_->cuMemObjGetDescriptor(ptr, size);
    if (rc == CU_OBJ_SUCCESS) {
        regions_.push_back({addr, size, 1});
    }
    return rc;
}
```

**But there is a better optimisation.**  In practice, the pages arrive in
ascending address order from a single contiguous buffer.  The very first
`registerMemory(page_0_addr, page_size)` call can be **speculatively
expanded** to cover the entire contiguous region that the first page
belongs to, up to the 4 GiB limit:

```cpp
cuObjErr_t CuObjTokenManager::registerMemory(void* ptr, size_t size) {
    std::lock_guard<std::mutex> lock(regions_mutex_);

    uintptr_t addr = reinterpret_cast<uintptr_t>(ptr);

    // Fast path: check if already within a registered region
    for (auto& region : regions_) {
        if (addr >= region.base && addr + size <= region.base + region.size) {
            region.refcount++;
            return CU_OBJ_SUCCESS;
        }
    }

    // New region: register with cuObject
    cuObjErr_t rc = client_->cuMemObjGetDescriptor(ptr, size);
    if (rc == CU_OBJ_SUCCESS) {
        regions_.push_back({addr, size, 1});
    }
    return rc;
}
```

With this approach:

- **First page**: `registerMemory(buf+0, page_size)` → calls
  `cuMemObjGetDescriptor` → registers `[buf, buf+page_size)`.
- **Second page**: `registerMemory(buf+page_size, page_size)` → not
  within first region → calls `cuMemObjGetDescriptor` again.

This still does N registrations.  To get O(1), the engine needs to know
the **pool size** at the time of the first registration.  Two strategies:

**Strategy A (engine-side, zero NIXL changes):** The Dell engine's
`registerMem` inspects the first DRAM/VRAM descriptor and calls
`cuMemObjGetDescriptor` with a **speculative large size** up to the 4 GiB
cuObject limit, rather than the page's own size:

```cpp
nixl_status_t S3DellObsObjEngineImpl::registerMem(
        const nixlBlobDesc& mem, const nixl_mem_t& nixl_mem,
        nixlBackendMD*& out) {
    if (nixl_mem == OBJ_SEG)
        return S3AccelObjEngineImpl::registerMem(mem, nixl_mem, out);

    // For DRAM/VRAM: register with cuObject (de-duplicated)
    cuObjErr_t rc = tokenMgr_->registerMemory(
        reinterpret_cast<void*>(mem.addr), mem.len);
    // ...
}
```

Since `CuObjTokenManager::registerMemory` checks if the address falls
within an already-registered region, the second through Nth pages are
instant no-ops — **provided the first registration covered them**.

**Strategy B (backend param, zero NIXL core changes):** Add an optional
backend parameter `rdma_pool_size` that tells the Dell engine the total
pool size.  When the first DRAM/VRAM `registerMem` arrives, the engine
registers `[mem.addr, mem.addr + rdma_pool_size)` (capped at 4 GiB) with
cuObject.  All subsequent pages within that range are de-duplicated.

```cpp
// In Dell engine constructor, read from backend params:
auto it = init_params->customParams->find("rdma_pool_size");
if (it != init_params->customParams->end())
    rdmaPoolSize_ = std::stoull(it->second);
```

```cpp
// In registerMem for DRAM/VRAM:
size_t reg_size = rdmaPoolSize_ > 0
    ? std::min(rdmaPoolSize_, (size_t)CUOBJ_MAX_MEMORY_REG_SIZE)
    : mem.len;
cuObjErr_t rc = tokenMgr_->registerMemory(
    reinterpret_cast<void*>(mem.addr), reg_size);
```

LMCache sets this in its backend params:
```python
backend_params = {
    # ... existing S3 params ...
    "rdma_pool_size": str(allocator.buffer_size),
}
```

**Strategy B is recommended** because:
1. It is explicit — the pool size is a known quantity at the LMCache side.
2. It does not require guessing or speculative over-registration.
3. It works correctly even if pages are not contiguous.
4. It requires zero changes to NIXL core or to the `registerMem` API.
5. The change on the LMCache side is one line in backend params.

#### Token generation with pool-relative offsets

Once the pool is registered once, token generation uses the pool base to
compute offsets, exactly as LMCache's connector does:

```cpp
std::string CuObjTokenManager::generateToken(
        void* data_ptr, size_t size, size_t offset, cuObjOpType_t op) {
    std::lock_guard<std::mutex> lock(regions_mutex_);

    uintptr_t addr = reinterpret_cast<uintptr_t>(data_ptr);

    // Find the region containing this address
    const RegisteredRegion* region = findContainingRegion(addr, size);
    if (!region)
        throw std::runtime_error("data_ptr not within a registered region");

    // Compute offset relative to the registered pool base
    size_t buffer_offset = (addr - region->base) + offset;

    char* desc_str = nullptr;
    cuObjErr_t rc = client_->cuMemObjGetRDMAToken(
        reinterpret_cast<void*>(region->base),  // pool base, not data_ptr
        size, buffer_offset, op, &desc_str);
    // ...
}
```

#### Deregistration

Reference counting per registered region.  Each `registerMemory` within
an existing region increments the count.  Each `deregisterMemory`
decrements it.  `cuMemObjPutDescriptor` is called only when the count
reaches zero.

#### Result

| Metric | Current (per-page) | Proposed (pool-level) |
|--------|--------------------|-----------------------|
| `cuMemObjGetDescriptor` calls | N (= buffer_size / page_size) | **1** |
| Registration time (2 GB, 1 MB pages) | 2,048 * ~0.5 ms = **~1 second** | **~0.5 ms** |
| Per-transfer `cuMemObjGetRDMAToken` | N/A (uses Pattern A callbacks) | 1 per page (lightweight, ~microseconds) |
| Data copies | 0 | 0 |

---

## 6. Detailed Component Design

### 6.1. CuObjTokenManager

**New file:** `cuobj_token_manager.h` / `cuobj_token_manager.cpp`

Thin RAII wrapper around `cuObjClient` for Pattern B token management.

```cpp
class CuObjTokenManager {
public:
    explicit CuObjTokenManager(cuObjProto_t proto = CUOBJ_PROTO_RDMA_DC_V1);
    ~CuObjTokenManager();

    // Non-copyable (owns cuObjClient instance)
    CuObjTokenManager(const CuObjTokenManager&) = delete;
    CuObjTokenManager& operator=(const CuObjTokenManager&) = delete;

    bool isConnected() const;

    // Memory registration (pool-aware, de-duplicating)
    cuObjErr_t registerMemory(void* ptr, size_t size);
    cuObjErr_t deregisterMemory(void* ptr, size_t size);

    // Token generation (thread-safe, computes offset from pool base)
    std::string generatePutToken(void* data_ptr, size_t size, size_t offset);
    std::string generateGetToken(void* data_ptr, size_t size, size_t offset);

private:
    std::string generateToken(void* data_ptr, size_t size,
                              size_t offset, cuObjOpType_t op);

    // Pool tracking: de-duplicate cuMemObjGetDescriptor calls
    struct RegisteredRegion {
        uintptr_t base;
        size_t    size;
        size_t    refcount;  // number of registerMemory calls within this region
    };
    const RegisteredRegion* findContainingRegion(uintptr_t ptr, size_t size) const;

    std::unique_ptr<cuObjClient> client_;
    std::vector<RegisteredRegion> regions_;
    mutable std::mutex regions_mutex_;  // protects regions_ for thread safety
};
```

**Constructor:**  Creates a `cuObjClient` with empty `CUObjIOOps` (no
callbacks -- Pattern B does not use `cuObjPut`/`cuObjGet`).  The empty
ops struct is required by the constructor signature but never invoked.

**`registerMemory`:**  Checks if `[ptr, ptr+size)` falls within an
already-registered region.  If yes, increments refcount and returns
immediately.  If not, calls `cuMemObjGetDescriptor(ptr, size)` and
records the region.  This de-duplicates the heavyweight NIC registration
when callers register many page-size chunks from a contiguous buffer.

**`deregisterMemory`:**  Decrements the refcount for the region containing
`[ptr, ptr+size)`.  Only calls `cuMemObjPutDescriptor(region.base)` when
refcount reaches zero.

**`generateToken`:**  Finds the registered region containing `data_ptr`,
computes `buffer_offset = (uintptr_t)data_ptr - region.base + offset`,
and calls `cuMemObjGetRDMAToken(region.base, size, buffer_offset, op,
&desc)`.  Copies the returned string, frees the library allocation via
`cuMemObjPutRDMAToken()`, and returns the copy.  This is the standard
Pattern B sequence from spec section 1.12.4, with pool-relative offset
computation borrowed from LMCache.

### 6.2. awsS3DellObsClient (rewritten)

**File:** `client.h` / `client.cpp`

Inherits from `awsS3AccelClient`.  Overrides the standard `iS3Client`
methods.

```cpp
class awsS3DellObsClient : public awsS3AccelClient {
public:
    awsS3DellObsClient(nixl_b_params_t* custom_params,
                        std::shared_ptr<CuObjTokenManager> token_mgr,
                        std::shared_ptr<Aws::Utils::Threading::PooledThreadExecutor>
                            executor = nullptr);

    void putObjectAsync(std::string_view key, uintptr_t data_ptr,
                        size_t data_len, size_t offset,
                        put_object_callback_t callback) override;

    void getObjectAsync(std::string_view key, uintptr_t data_ptr,
                        size_t data_len, size_t offset,
                        get_object_callback_t callback) override;

private:
    std::shared_ptr<CuObjTokenManager> tokenMgr_;
};
```

**`putObjectAsync`:**
1. Validate `data_len > 0` and `offset == 0` (Dell PUT does not support
   offsets).  Fail-fast via `callback(false)` for invalid parameters.
2. **Try RDMA path:**
   a. Call `tokenMgr_->generatePutToken(data_ptr, data_len, 0)`.
   b. Build `Aws::S3::Model::PutObjectRequest` with `x-rdma-info` header
      set to the token and `ContentLength(0)`.
   c. Call `s3Client_->PutObjectAsync(request, ...)` with the
      success/failure callback.
3. **On exception (per-request fallback):** catch the exception, log a
   warning, and delegate to `awsS3AccelClient::putObjectAsync(...)` which
   sends the data via standard HTTP body.  The caller never sees the
   failure.

```cpp
void awsS3DellObsClient::putObjectAsync(
        std::string_view key, uintptr_t data_ptr, size_t data_len,
        size_t offset, put_object_callback_t callback) {
    if (data_len == 0) { callback(false); return; }
    if (offset != 0)   { callback(false); return; }

    try {
        std::string token = tokenMgr_->generatePutToken(
            reinterpret_cast<void*>(data_ptr), data_len, 0);

        Aws::S3::Model::PutObjectRequest request;
        request.WithBucket(bucketName_).WithKey(Aws::String(key));
        request.SetAdditionalCustomHeaderValue("x-rdma-info", token);
        request.SetContentLength(0);

        s3Client_->PutObjectAsync(request,
            [callback](auto*, auto&, const auto& outcome, auto&) {
                callback(outcome.IsSuccess());
            }, nullptr);
    } catch (const std::exception& e) {
        // HTTP fallback is only safe for CPU-addressable memory
        auto mem_type = cuObjClient::getMemoryType(
            reinterpret_cast<const void*>(data_ptr));
        if (mem_type == CUOBJ_MEMORY_CUDA_DEVICE) {
            NIXL_ERROR << "RDMA put failed for VRAM, cannot fall back: "
                       << e.what();
            callback(false);
            return;
        }
        NIXL_WARN << "RDMA put failed, falling back to HTTP: " << e.what();
        awsS3AccelClient::putObjectAsync(key, data_ptr, data_len,
                                          offset, callback);
    }
}
```

**`getObjectAsync`:**
1. Validate `data_len > 0` and guard against `offset + data_len` overflow.
2. **Try RDMA path:**
   a. Call `tokenMgr_->generateGetToken(data_ptr, data_len, offset)`.
   b. Build `Aws::S3::Model::GetObjectRequest` with `x-rdma-info` header
      and byte range.
   c. Call `s3Client_->GetObjectAsync(request, ...)`.
3. **On exception (per-request fallback):** check memory type; fall back to
   HTTP for DRAM, fail for VRAM.

```cpp
void awsS3DellObsClient::getObjectAsync(
        std::string_view key, uintptr_t data_ptr, size_t data_len,
        size_t offset, get_object_callback_t callback) {
    if (data_len == 0) { callback(false); return; }
    if ((data_len > 0) && (offset > (SIZE_MAX - (data_len - 1)))) {
        callback(false); return;
    }

    try {
        std::string token = tokenMgr_->generateGetToken(
            reinterpret_cast<void*>(data_ptr), data_len, offset);

        Aws::S3::Model::GetObjectRequest request;
        request.WithBucket(bucketName_).WithKey(Aws::String(key))
               .WithRange(absl::StrFormat("bytes=%zu-%zu",
                          offset, offset + data_len - 1));
        request.SetAdditionalCustomHeaderValue("x-rdma-info", token);

        s3Client_->GetObjectAsync(request,
            [callback](auto*, auto&, const auto& outcome, auto&) {
                callback(outcome.IsSuccess());
            }, nullptr);
    } catch (const std::exception& e) {
        auto mem_type = cuObjClient::getMemoryType(
            reinterpret_cast<const void*>(data_ptr));
        if (mem_type == CUOBJ_MEMORY_CUDA_DEVICE) {
            NIXL_ERROR << "RDMA get failed for VRAM, cannot fall back: "
                       << e.what();
            callback(false);
            return;
        }
        NIXL_WARN << "RDMA get failed, falling back to HTTP: " << e.what();
        awsS3AccelClient::getObjectAsync(key, data_ptr, data_len,
                                          offset, callback);
    }
}
```

**Fallback semantics:** The `try/catch` boundary is drawn tightly around
the RDMA-specific code (token generation + RDMA request construction).
The catch clause checks `cuObjClient::getMemoryType()` to determine if
the buffer is CPU-addressable before falling back to HTTP.  For DRAM,
the parent's `PreallocatedStreamBuf` wraps the buffer zero-copy.  For
VRAM, fallback is unsafe so the request fails immediately.

### 6.3. S3DellObsObjEngineImpl (simplified)

**File:** `engine_impl.h` / `engine_impl.cpp`

```cpp
class S3DellObsObjEngineImpl : public S3AccelObjEngineImpl {
public:
    explicit S3DellObsObjEngineImpl(const nixlBackendInitParams* init_params);
    S3DellObsObjEngineImpl(const nixlBackendInitParams* init_params,
                           std::shared_ptr<iS3Client> s3_client);

    nixl_mem_list_t getSupportedMems() const override {
        return {OBJ_SEG, DRAM_SEG, VRAM_SEG};
    }

    nixl_status_t registerMem(const nixlBlobDesc& mem,
                              const nixl_mem_t& nixl_mem,
                              nixlBackendMD*& out) override;

    nixl_status_t deregisterMem(nixlBackendMD* meta) override;

    // prepXfer  -- INHERITED from DefaultObjEngineImpl
    // postXfer  -- INHERITED from DefaultObjEngineImpl
    // checkXfer -- INHERITED from DefaultObjEngineImpl
    // releaseReqH -- INHERITED from DefaultObjEngineImpl

protected:
    iS3Client* getClient() const override;

private:
    std::shared_ptr<iS3Client> s3Client_;
    std::shared_ptr<CuObjTokenManager> tokenMgr_;
};
```

**`registerMem`:**
- `OBJ_SEG`: delegates to `S3AccelObjEngineImpl::registerMem` (key mapping).
- `DRAM_SEG` / `VRAM_SEG`: calls `tokenMgr_->registerMemory(ptr, size)`,
  stores a `nixlDellMemMetadata` with the address for later deregistration.

**`deregisterMem`:**
- If metadata is `nixlDellMemMetadata`: calls `tokenMgr_->deregisterMemory`.
- Otherwise: delegates to parent for OBJ_SEG cleanup.

**Dell-specific metadata (in anonymous namespace):**

```cpp
class nixlDellMemMetadata : public nixlBackendMD {
public:
    nixlDellMemMetadata(nixl_mem_t type, uintptr_t a, size_t l)
        : nixlBackendMD(true), memType(type), addr(a), len(l) {}
    nixl_mem_t memType;
    uintptr_t addr;
    size_t len;
};
```

### 6.4. Interaction with DefaultObjEngineImpl::prepXfer

The parent's `prepXfer` validates that `local.getType() == DRAM_SEG`.
Since the Dell engine also supports `VRAM_SEG`, the parent's validation
function (`isValidPrepXferParams` in the anonymous namespace of
`s3/engine_impl.cpp`) would reject VRAM.

**Resolution:** The parent's validation already lives in an anonymous
namespace helper.  One of these approaches works:

- **(Preferred)** Extend the parent's check to accept local segment types
  reported by `getSupportedMems()`, OR
- Override `prepXfer` in the Dell engine with a one-line change to the
  validation, delegating everything else to the parent.

The preferred approach is a small, non-disruptive change to the parent's
validation function to accept any local memory type that the engine
reports as supported.  This benefits future vendor engines as well.

---

## 7. End-to-End Transfer Flow

### Write (PUT)

```
User                  nixlObjEngine    DefaultObjEngineImpl      awsS3DellObsClient    CuObjTokenManager
 |                         |                   |                        |                      |
 |-- registerMem(VRAM) --> |                   |                        |                      |
 |                         |-- registerMem --> |                        |                      |
 |                         |       Dell: tokenMgr_->registerMemory(ptr, size) ------------->  |
 |                         |                   |                        |   cuMemObjGetDescr.. |
 |                         |                   |                        |                      |
 |-- prepXfer(WRITE) ----> |                   |                        |                      |
 |                         |-- prepXfer -----> |                        |                      |
 |                         |    INHERITED: validate, create req handle  |                      |
 |                         |                   |                        |                      |
 |-- postXfer(WRITE) ----> |                   |                        |                      |
 |                         |-- postXfer -----> |                        |                      |
 |                         |    INHERITED: for each descriptor:         |                      |
 |                         |      getClient()->putObjectAsync(key, ptr, len, 0, cb)           |
 |                         |                   | ---- putObjectAsync -> |                      |
 |                         |                   |          try {                                |
 |                         |                   |            1. generatePutToken(ptr,len,0) --> |
 |                         |                   |            |        cuMemObjGetRDMAToken      |
 |                         |                   |            |        cuMemObjPutRDMAToken      |
 |                         |                   |            | <---- token string ------------ |
 |                         |                   |            2. PutObjectRequest + x-rdma-info  |
 |                         |                   |            3. ContentLength(0)                |
 |                         |                   |            4. s3Client_->PutObjectAsync(req)  |
 |                         |                   |            |                                  |
 |                         |                   |            | ===== RDMA_READ by server =====  |
 |                         |                   |          }                                    |
 |                         |                   |          catch (...) {                        |
 |                         |                   |            NIXL_WARN << "fallback to HTTP"    |
 |                         |                   |            awsS3AccelClient::putObjectAsync() |
 |                         |                   |            | ===== HTTP body upload =====     |
 |                         |                   |          }                                    |
 |                         |    bridge: callback(success) --> promise.set_value(NIXL_SUCCESS)  |
 |                         |                   |                        |                      |
 |-- checkXfer ----------> |                   |                        |                      |
 |                         |-- checkXfer ----> |                        |                      |
 |                         |    INHERITED: poll futures                  |                      |
 |                         |                   |                        |                      |
 |-- releaseReqH --------> |                   |                        |                      |
 |                         |-- releaseReqH --> |                        |                      |
 |                         |    INHERITED: delete handle                |                      |
```

### Read (GET)

Identical to Write except:
- `getObjectAsync` is called instead of `putObjectAsync`.
- Token is generated with `generateGetToken(ptr, len, offset)`.
- The S3 request includes a byte range header.
- The server performs `RDMA_WRITE` into the client buffer.
- On fallback, the parent's `getObjectAsync` downloads via HTTP body.

---

## 8. File Change Summary

| File | Action | Lines (approx.) | Notes |
|------|--------|-----------------|-------|
| `s3_accel/dell/cuobj_token_manager.h` | **NEW** | ~40 | RAII wrapper, Pattern B API |
| `s3_accel/dell/cuobj_token_manager.cpp` | **NEW** | ~60 | Token generation, memory registration |
| `s3_accel/dell/client.h` | **REWRITE** | ~30 (was 73) | Remove `iDellS3RdmaClient` inheritance, override standard `iS3Client` |
| `s3_accel/dell/client.cpp` | **REWRITE** | ~80 (was 176) | Pattern B token injection in `putObjectAsync`/`getObjectAsync` |
| `s3_accel/dell/engine_impl.h` | **SIMPLIFY** | ~40 (was 154) | Remove `prepXfer`/`postXfer`/`checkXfer`/`releaseReqH` overrides |
| `s3_accel/dell/engine_impl.cpp` | **SIMPLIFY** | ~120 (was 603) | Only `registerMem`/`deregisterMem`/constructors/`getClient` |
| `s3_accel/dell/rdma_interface.h` | **DELETE** | -57 | `iDellS3RdmaClient` no longer needed |
| `meson.build` | **MINOR** | ~2 line delta | Replace `rdma_interface.h` with `cuobj_token_manager.{h,cpp}` |
| `s3/engine_impl.cpp` | **MINOR** | ~3 line delta | Extend `isValidPrepXferParams` to accept VRAM_SEG for engines that support it |
| `obj_backend.cpp` | **NO CHANGE** | 0 | Factory logic unchanged |
| `engine_utils.h` | **NO CHANGE** | 0 | `isDellOBSRequested` unchanged |
| `test/gtest/unit/obj/obj.cpp` | **SIMPLIFY** | ~-50 | Remove `mockDellS3Client`; standard `mockS3Client` works |

**Net:** ~370 lines of Dell code (was ~1,100) -- roughly 70% reduction.

---

## 9. Migration Checklist

1. Create `cuobj_token_manager.h` and `cuobj_token_manager.cpp`.
2. Rewrite `client.h` / `client.cpp` to override `putObjectAsync` /
   `getObjectAsync` with Pattern B token injection.
3. Delete `rdma_interface.h`.
4. Simplify `engine_impl.h` / `engine_impl.cpp` -- remove all transfer
   lifecycle overrides, keep `registerMem` / `deregisterMem` /
   `getSupportedMems` / `getClient` / constructors.
5. Update `meson.build` to list the new files and remove `rdma_interface.h`.
6. Extend `isValidPrepXferParams` in `s3/engine_impl.cpp` to accept
   VRAM_SEG for engines that report it in `getSupportedMems()`.
7. Simplify tests: remove `mockDellS3Client`, use standard `mockS3Client`.
8. Verify: all existing tests pass, no NIXL public API changes.
