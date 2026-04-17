# OPcache ZTS Thread-Safety Fix — Implementation Plan

## Problem Statement

OPcache's shared memory structures (`ZCSG(hash)`, interned strings buffer) are accessed lock-free by concurrent reader threads but modified destructively by `opcache_reset()` / `opcache_invalidate()` without proper synchronization. This causes `zend_mm_heap corrupted` crashes under concurrent load in both ZTS (FrankenPHP, parallel) and FPM multi-process configurations.

The bug has been open since 2022 (php-src#8739) with escalating reports as FrankenPHP adoption grows. A WIP fix (PR #14803) was rejected by Dmitry Stogov because it introduced global locking on every cache lookup, violating OPcache's lock-free read design.

### Upstream References

| Reference | Description |
|---|---|
| [php-src#8739](https://github.com/php/php-src/issues/8739) | Original report — string corruption on deployments with php-fpm 8.1.6 (2022) |
| [php-src#14471](https://github.com/php/php-src/issues/14471) | Reproducer — `opcache_reset()` crash under high load in FPM (2024) |
| [php-src#18517](https://github.com/php/php-src/issues/18517) | FrankenPHP-specific — opcache_reset memory corruption in ZTS under concurrency |
| [php-src PR #14803](https://github.com/php/php-src/pull/14803) | WIP fix by ndossche — rejected, needs lock-free approach |
| [frankenphp#1737](https://github.com/dunglas/frankenphp/issues/1737) | WordPress multisite crashes every ~8 hours |
| [frankenphp#2265](https://github.com/dunglas/frankenphp/issues/2265) | Worker restart via admin API corrupts heap |
| [frankenphp#2170](https://github.com/dunglas/frankenphp/issues/2170) | Symfony deploy + cache:clear crash |

### Our Production Backtrace

From a FrankenPHP production server (PHP 8.4.20 ZTS, glibc, x86_64) running WordPress multisite:

```
#25 frankenphp.c:1088        php_thread — executing wp-login.php
#24 php_execute_script       — running a PHP script
#19 ZEND_HANDLE_EXCEPTION    — handling an exception
#17 zend_leave_helper        — leaving a scope
#16 zend_detach_symbol_table — detaching symbol table during scope exit
#15 zend_hash_del            — deleting from hash table (zend_hash.c:1562)
#14 zend_string_release      — freeing a zend_string
#13 _efree                   — Zend memory free
#12 zend_mm_free_heap        — heap corruption detected (ptr=0x42570bd0, heap=0x733e1c600040)
#11 zend_mm_panic            — "zend_mm_heap corrupted"
#10 abort → SIGABRT
```

The pointer `0x42570bd0` is outside the thread's heap at `0x733e1c600040` — it points to shared OPcache memory that was freed by a concurrent `opcache_reset()` while this thread still held a reference to it.

**Key observation:** Setting `USE_ZEND_ALLOC=0` (system malloc) eliminates the crash entirely. System malloc handles cross-thread frees gracefully; Zend's per-thread heap allocator does not.

---

## Root Cause Analysis

### Race Condition 1: Hash Table Lookup vs Reset

In `ext/opcache/ZendAccelerator.c`, `persistent_compile_file()` (line ~2090) and `persistent_zend_resolve_path()` (line ~2586) perform a two-step operation:

```c
// Step 1: Find entry in shared hash table (lock-free)
bucket = zend_accel_hash_find_entry(&ZCSG(hash), key);

// <<< RACE WINDOW: another thread calls opcache_reset() here >>>
// opcache_reset() calls zend_accel_hash_clean() which does:
//   memset(hash_table, 0, sizeof(entry*) * max_entries)
// This zeroes out hash_table entries while bucket still points to old data

// Step 2: Use the bucket (now dangling)
if (bucket) {
    zend_accel_add_key(key, bucket);  // writes to freed/zeroed memory
}
```

`zend_accel_hash_clean()` (zend_accelerator_hash.c:30-35) is a simple memset:

```c
void zend_accel_hash_clean(zend_accel_hash *accel_hash)
{
    accel_hash->num_entries = 0;
    accel_hash->num_direct_entries = 0;
    memset(accel_hash->hash_table, 0, sizeof(zend_accel_hash_entry *)*accel_hash->max_num_entries);
}
```

No fence, no epoch, no indication to readers that the table was invalidated between their find and their use of the result.

### Race Condition 2: Interned String Lifetime

When `opcache_reset()` calls `accel_interned_strings_restore_state()` (line ~2750), it resets the interned strings buffer to its initial state. Any thread still holding a pointer to an interned string (e.g., a `zend_string*` from a cached script's filename) now has a dangling pointer. When that thread later calls `zend_string_release()`, it tries to free memory that belongs to the (now-reset) shared buffer, causing heap corruption.

This is exactly what our backtrace shows: `zend_string_release` on a pointer (`0x42570bd0`) that's in the shared memory region, not the thread's private heap.

### Race Condition 3: Script Corruption Flag

The `persistent_script->corrupted` flag is checked without synchronization. A script can be marked corrupted by one thread while another thread is mid-execution of the same script's op_array. The executing thread may encounter partially-invalidated data structures.

### Why PR #14803 Was Rejected

PR #14803 by @ndossche wrapped the find+add sequences with `zend_shared_alloc_lock()`:

```c
zend_shared_alloc_lock();
bucket = zend_accel_hash_find_entry(&ZCSG(hash), key);
if (bucket) {
    zend_accel_add_key(key, bucket);
}
zend_shared_alloc_unlock();
```

Dmitry Stogov's review comment:
> "Please think about a different solution. This one should make a big slowdown.
> opcache should be able to do lock-free cache lookups.
> Probably, the modification of `ZCSG(hash)` should be done more careful
> (to keep consistency for concurrent readers)"

**The constraint is clear: readers must remain lock-free. The fix must be on the writer side.**

### Why `USE_ZEND_ALLOC=0` Works

With system malloc (`USE_ZEND_ALLOC=0`), freed memory goes to the OS allocator which handles cross-thread frees gracefully. The dangling pointer still exists, but:
1. System malloc doesn't panic on double-free or cross-arena free — it just handles it
2. The freed memory isn't immediately reused within a thread-local heap slab
3. The corruption is "absorbed" by the more forgiving allocator

This is a workaround, not a fix. The dangling pointers still exist.

---

## Proposed Solution: Epoch-Based Reclamation (EBR)

### Design Principles

1. **Lock-free reads** — readers never acquire a lock (Dmitry's constraint)
2. **Safe reclamation** — shared memory is only freed after all readers that could reference it have completed
3. **Bounded memory** — deferred frees don't accumulate unboundedly
4. **Minimal overhead** — per-request cost is a single atomic increment/decrement

### How EBR Works

Epoch-based reclamation is a well-proven pattern from lock-free data structure research (Fraser 2004, Hart et al. 2007). It's used in the Linux kernel (RCU), FreeBSD, Concurrency Kit, and Crossbeam (Rust).

The key insight: instead of preventing concurrent access to shared data (locking), we prevent *reclamation* of shared data until all readers that might reference it have finished.

```
Global epoch counter: E (monotonically increasing)

Reader protocol:
  1. On request start:  thread_epoch[tid] = E  (atomic store)
  2. Execute request (may read shared OPcache data)
  3. On request end:    thread_epoch[tid] = INACTIVE  (atomic store)

Writer protocol (opcache_reset):
  1. Increment E to E+1
  2. Mark old data as "retired at epoch E"
  3. Wait until min(thread_epoch[all active threads]) > E
     (i.e., all threads that were active during epoch E have completed)
  4. Now safe to free/reuse the old data
```

### Adaptation to OPcache Architecture

OPcache doesn't allocate/free individual entries — it uses a contiguous shared memory segment (`zend_shared_alloc`) with a watermark allocator. `opcache_reset()` resets the watermark to reclaim all memory at once. This means we can't do per-object deferred freeing; we need to defer the entire reset operation.

**Proposed approach: Double-buffering with epoch guard**

Instead of resetting the shared memory in-place while readers are active, we:

1. **Mark the current generation as "draining"** — new requests will not cache into it
2. **Allocate new scripts into a fresh generation** — the watermark is reset for new allocations, but old data is preserved
3. **Wait for all active readers to complete** — using epoch tracking
4. **Only then reclaim the old generation** — memset the hash table, restore interned strings

This is conceptually similar to how RCU "grace periods" work in the Linux kernel.

### Implementation Detail

#### New Data Structures

In `ext/opcache/ZendAccelerator.h`:

```c
/* Per-thread epoch tracking for safe reclamation */
typedef struct _zend_accel_epoch {
    volatile uint64_t epoch;        /* Current epoch this thread is in, or UINT64_MAX if inactive */
    char padding[56];               /* Pad to cache line to avoid false sharing */
} zend_accel_epoch;

/* Add to ZCSG (shared globals): */
// volatile uint64_t current_epoch;           /* Global epoch counter */
// volatile uint64_t drain_epoch;             /* Epoch at which reset was requested */
// volatile bool     reset_deferred;          /* True if a reset is pending drain completion */
```

In `ZCG` (per-thread/process globals):

```c
// uint64_t local_epoch;  /* Snapshot of global epoch at request start */
```

#### Modified Request Lifecycle

**Request start** (`accel_activate`, called per-request):

```c
void accel_activate(void)
{
    /* Enter epoch — publish that this thread is reading shared data */
    ZCG(local_epoch) = __atomic_load_n(&ZCSG(current_epoch), __ATOMIC_ACQUIRE);
    __atomic_store_n(&ZCSG(epochs)[thread_id].epoch, ZCG(local_epoch), __ATOMIC_RELEASE);

    /* ... existing activation code ... */
}
```

**Request end** (`accel_deactivate`, called per-request):

```c
void accel_deactivate(void)
{
    /* ... existing deactivation code ... */

    /* Leave epoch — this thread no longer references shared data */
    __atomic_store_n(&ZCSG(epochs)[thread_id].epoch, UINT64_MAX, __ATOMIC_RELEASE);

    /* Check if we can complete a deferred reset */
    if (ZCSG(reset_deferred) && can_complete_reset()) {
        complete_deferred_reset();
    }
}
```

#### Modified `opcache_reset()` Flow

**Current flow** (racy):
```
opcache_reset() → zend_accel_hash_clean() → memset → accel_interned_strings_restore_state()
                  ↑ readers still active, using pointers to this memory
```

**Proposed flow** (safe):
```
opcache_reset():
  1. ZCSG(drain_epoch) = ZCSG(current_epoch)
  2. ZCSG(current_epoch)++           // New requests enter new epoch
  3. ZCSG(reset_deferred) = true
  4. ZCSG(accelerator_enabled) = false  // Stop caching new scripts (existing behavior)
  // Do NOT call zend_accel_hash_clean() yet

complete_deferred_reset() [called from accel_deactivate when safe]:
  if (min_active_epoch() > ZCSG(drain_epoch)):
    // All threads from the old epoch have completed
    zend_accel_hash_clean(&ZCSG(hash))
    accel_interned_strings_restore_state()
    zend_shared_alloc_restore_state()
    ZCSG(reset_deferred) = false
    ZCSG(accelerator_enabled) = true
```

The `min_active_epoch()` function scans all thread epoch slots:

```c
static uint64_t min_active_epoch(void)
{
    uint64_t min = UINT64_MAX;
    for (int i = 0; i < max_threads; i++) {
        uint64_t e = __atomic_load_n(&ZCSG(epochs)[i].epoch, __ATOMIC_ACQUIRE);
        if (e < min) min = e;
    }
    return min;
}
```

#### Modified `opcache_invalidate()` Flow

`opcache_invalidate()` for a single file is less destructive but has the same race. The fix is simpler:

1. Set `persistent_script->corrupted = true` (atomic store)
2. Don't unlink the hash entry immediately
3. Defer the actual memory reclamation to the next safe epoch

Since invalidating a single script doesn't reset the entire SHM watermark, we can mark individual scripts as "retired" and skip them in lookups without needing to free their memory immediately. The memory is reclaimed on the next full `opcache_reset()` (which already resets the watermark).

### ZTS vs FPM Considerations

**ZTS (FrankenPHP, parallel):** Threads share address space. The epoch array lives in shared memory alongside `ZCSG`. Thread ID can be the PHP thread index (already tracked in FrankenPHP as `thread_index`). Atomic operations use `__atomic_*` builtins.

**FPM:** Processes share memory via mmap. The epoch array lives in the same shared memory segment as `ZCSG`. Process ID mapping needs a slot allocation scheme (e.g., `getpid() % max_children` or an atomic slot allocator). Atomics work across processes on shared mmap'd memory.

**Both:** The per-request enter/leave cost is two atomic stores (one cache line write each). The deferred reset scan is O(max_threads) but only runs at request end when a reset is pending — not on every request.

---

## Files to Modify

| File | Changes |
|---|---|
| `ext/opcache/ZendAccelerator.h` | Add epoch structures to `zend_accel_shared_globals`, per-thread epoch fields to `zend_accel_globals` |
| `ext/opcache/ZendAccelerator.c` | Modify `accel_activate()`, `accel_deactivate()`, restart logic (lines 2730-2770), `persistent_compile_file()`, `persistent_zend_resolve_path()` |
| `ext/opcache/zend_accelerator_hash.c` | Add memory barrier after `zend_accel_hash_clean()` memset |
| `ext/opcache/zend_accelerator_module.c` | Modify `ZEND_FUNCTION(opcache_reset)` and `ZEND_FUNCTION(opcache_invalidate)` to use deferred path |
| `ext/opcache/zend_accelerator_util_funcs.c` | Add epoch check in `zend_accel_load_script()` before using cached class entries |
| `ext/opcache/zend_shared_alloc.h` | Add epoch-related declarations |
| `ext/opcache/zend_shared_alloc.c` | Allocate epoch array in shared memory during initialization |

## Implementation Phases

### Phase 1: Epoch Infrastructure

1. Define the `zend_accel_epoch` structure and add it to shared globals
2. Allocate the epoch array in `zend_shared_alloc_startup()`
3. Implement `accel_epoch_enter()`, `accel_epoch_leave()`, `accel_min_active_epoch()`
4. Wire enter/leave into `accel_activate()` and `accel_deactivate()`
5. Unit test: verify epoch tracking with concurrent threads

### Phase 2: Deferred Reset

1. Add `drain_epoch`, `reset_deferred` fields to shared globals
2. Modify the restart logic (ZendAccelerator.c:2730-2770) to defer the actual cleanup
3. Implement `complete_deferred_reset()` called from `accel_deactivate()`
4. Test with the reproducer from php-src#14471: `opcache_reset()` under concurrent load
5. Test with FrankenPHP reproducer from frankenphp#1737: WordPress reinstall under traffic

### Phase 3: Safe Invalidation

1. Modify `opcache_invalidate()` to set `corrupted` atomically without unlinking
2. Add epoch check before using cached script data in `persistent_compile_file()`
3. Defer hash entry cleanup to next full reset
4. Test with WordPress `wp_opcache_invalidate()` under concurrent traffic

### Phase 4: FPM Support

1. Implement process-based slot allocation for the epoch array
2. Handle process death (stale epoch entries) — watchdog timer or `SIGCHLD` cleanup
3. Test with the FPM reproducer from php-src#14471

---

## Testing Strategy

### Reproducer Scripts

**Minimal reproducer** (from php-src#14471):
```php
<?php
class T {}
new T;
opcache_reset();
```
Run under `ab -c 24 -n 200000` or `vegeta attack -duration=3s -rate=400/1s`.

**WordPress reproducer** (from frankenphp#1737):
Navigate to wp-admin → Updates → "Reinstall version X.X" under concurrent traffic.

**FrankenPHP ZTS reproducer** (from frankenphp#18517):
Use the reproducer at https://github.com/AlliBalliBaba/zts-opcache-reset-reproducer

### Verification

1. **Valgrind:** Run FPM under valgrind with the minimal reproducer — should show zero invalid reads/writes
2. **ASAN:** Build with `--enable-address-sanitizer` — should show zero heap-use-after-free
3. **TSAN:** Build with `--enable-thread-sanitizer` (ZTS) — should show zero data races on ZCSG fields
4. **Stress test:** 10-minute sustained load with periodic `opcache_reset()` — zero crashes
5. **Performance:** Benchmark with `opcache_reset()` disabled to measure epoch enter/leave overhead — should be <1% (two atomic stores per request)

### Regression Tests

Add to `ext/opcache/tests/`:
- `reset_under_concurrency_zts.phpt` — ZTS-specific test with parallel threads
- `reset_under_concurrency_fpm.phpt` — FPM-specific test with concurrent requests
- `invalidate_under_concurrency.phpt` — Single-file invalidation under load

---

## Risk Assessment

| Risk | Mitigation |
|---|---|
| Deferred reset delays memory reclamation | Bounded: reset completes as soon as all active requests finish (typically <1s) |
| Epoch array size limits max threads | Size to `opcache.max_threads` (new INI) or default to 256 slots |
| Process death leaves stale epoch entries (FPM) | Scan for dead PIDs in `min_active_epoch()`; treat dead entries as inactive |
| Atomic operations not available on all platforms | Use `__atomic_*` builtins (GCC 4.7+, Clang 3.1+); fall back to `zend_atomic.h` |
| Double-reset before drain completes | Queue the second reset; only one can be draining at a time |

---

## PR Strategy

**Target branch:** `PHP-8.4` (backportable to 8.2, 8.3)

**PR title:** `Fix opcache_reset()/opcache_invalidate() races using epoch-based reclamation`

**Key selling points for reviewers:**
1. **Lock-free reads preserved** — readers do two atomic stores per request (enter/leave), never block
2. **Writers wait, not readers** — reset is deferred until all pre-epoch readers complete
3. **Proven pattern** — epoch-based reclamation is used in Linux kernel (RCU), FreeBSD, and production lock-free data structures
4. **Complete fix** — addresses all three race conditions (hash lookup, interned strings, script corruption flag)
5. **Includes reproducers** — concrete test cases from php-src#14471 and FrankenPHP production

**Reference Dmitry's constraint explicitly:** The PR description should quote his review comment from PR #14803 and explain how this approach satisfies "lock-free cache lookups."

---

## Open Questions for PHP Core Team

1. **Max thread/process slots:** What's a reasonable default for the epoch array size? FrankenPHP can have 100+ threads; FPM can have 500+ children. Suggest 256 default, configurable via `opcache.max_concurrent_readers`.

2. **Shared memory layout:** The epoch array needs to live in the same shared memory segment as `ZCSG`. Is `zend_shared_alloc()` the right allocator, or should it be a separate mmap region to avoid fragmenting the script cache?

3. **Preload interaction:** `ZCSG(preload_script)` is handled specially during restart. Does the epoch guard need special handling for preloaded scripts, or are they always resident?

4. **JIT interaction:** `zend_jit_restart()` is called during reset. Does JIT compiled code hold references to interned strings that would be invalidated? If so, the JIT restart also needs to be deferred.

5. **Windows:** The current restart logic uses `fcntl` file locks on Unix and `INCREMENT`/`DECREMENT` on Windows. The epoch approach uses atomics which work on both. Should we unify the platform-specific restart paths?
