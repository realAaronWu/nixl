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
   client's async method calls `callback(false)` immediately.  The parent's
   future/promise bridge translates this to `NIXL_ERR_BACKEND`.  No special
   error path is needed in the engine.

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

    // Memory registration
    cuObjErr_t registerMemory(void* ptr, size_t size);
    cuObjErr_t deregisterMemory(void* ptr);

    // Token generation (thread-safe)
    std::string generatePutToken(void* base_ptr, size_t size, size_t offset);
    std::string generateGetToken(void* base_ptr, size_t size, size_t offset);

private:
    std::string generateToken(void* base_ptr, size_t size,
                              size_t offset, cuObjOpType_t op);
    std::unique_ptr<cuObjClient> client_;
};
```

**Constructor:**  Creates a `cuObjClient` with empty `CUObjIOOps` (no
callbacks -- Pattern B does not use `cuObjPut`/`cuObjGet`).  The empty
ops struct is required by the constructor signature but never invoked.

**`generateToken`:**  Calls `cuMemObjGetRDMAToken()`, copies the returned
string, frees the library allocation via `cuMemObjPutRDMAToken()`, and
returns the copy.  This is the standard Pattern B sequence from spec
section 1.12.4.

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
   offsets).
2. Call `tokenMgr_->generatePutToken(data_ptr, data_len, 0)`.
3. Build `Aws::S3::Model::PutObjectRequest` with `x-rdma-info` header
   set to the token and `ContentLength(0)`.
4. Call `s3Client_->PutObjectAsync(request, ...)` with the
   success/failure callback.

**`getObjectAsync`:**
1. Validate `data_len > 0` and guard against `offset + data_len` overflow.
2. Call `tokenMgr_->generateGetToken(data_ptr, data_len, offset)`.
3. Build `Aws::S3::Model::GetObjectRequest` with `x-rdma-info` header
   and byte range.
4. Call `s3Client_->GetObjectAsync(request, ...)`.

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
 |                         |                   |          1. generatePutToken(ptr, len, 0) --> |
 |                         |                   |          |        cuMemObjGetRDMAToken        |
 |                         |                   |          |        cuMemObjPutRDMAToken        |
 |                         |                   |          | <---- token string --------------- |
 |                         |                   |          2. PutObjectRequest + x-rdma-info    |
 |                         |                   |          3. ContentLength(0)                  |
 |                         |                   |          4. s3Client_->PutObjectAsync(req)    |
 |                         |                   |          |                                    |
 |                         |                   |          |  ===== RDMA_READ by server =====   |
 |                         |                   |          |                                    |
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
