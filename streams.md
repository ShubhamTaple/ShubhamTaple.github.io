# Redesigning idempotent stream keys migration logic for asmBgTrim

Hi all, here I would be documenting the idea and implementation logic of one of my contributions to Redis OSS.

## Background
While reading a PR (https://github.com/redis/redis/pull/14897) which implemented the DB lifecycle of idempotent stream keys by Sergei Georgiev (Principal Software Engineer at Redis), I came across several doubts, two of which turned out to be things that could be made better. One of them is the current topic of discussion (https://github.com/redis/redis/pull/15000), and the other was to have a new dictionary type with only keys and no values (a set, essentially) for idempotent (IDMP) stream keys — this one I documented separately on my Medium blog.

## The idea
In Sergei's PR, there was a helper function exposed from the core `db.c` file called `streamMoveIdmpKeys()`, which helped migrate IDMP stream keys from one dictionary to another during atomic slot migration.

**Function signature:**
`void streamMoveIdmpKeys(dict *src, dict *dst, int slot)`

`streamMoveIdmpKeys()` took an `int slot` parameter, iterated over IDMP stream keys in the source dict, and moved the keys belonging to that `slot` into the destination dict.

```c
typedef struct slotRange {
    unsigned short start, end;
} slotRange;

typedef struct slotRangeArray {
    int num_ranges;
    slotRange ranges[];
} slotRangeArray;
```

atp (having known that a `struct` called `slotRangeArray` already exists, which takes care of ranges of slots rather than individual slots) `streamMoveIdmpKeys()` being written for each individual slot felt off to me.

Tracing where the function was used made it clear it's only called from the ASM background-trim logic, to clean up expired IDMP streams. The chances of this activity happening on individual slots is much lower than it happening on a range of slots — that's how clusters actually own slots. So I modified the code first (which worked), raised the PR, and then filed an issue describing the broader improvement (https://github.com/redis/redis/issues/15001).

## Implementation

*I humbly believe it was all about the idea, and not about implementation.*

The aim (as simple as it is) was to convert the function from this form:
`void streamMoveIdmpKeys(dict *src, dict *dst, int slot)`

to this form:
`void streamMoveIdmpKeys(dict *src, dict *dst, slotRangeArray *slots)`

— calling it once for the whole slot range instead of once per **individual slot** in the ASM bgtrim logic, and writing a couple of tests to verify the logic.

```c
/* Move entries whose robj keys belong to the given slotRangeArray from src dict to dst.
 * Matching entries are removed from src and added to dst. */
void streamMoveIdmpKeys(dict *src, dict *dst, slotRangeArray *slots) {
    if (dictSize(src) == 0) return;

    /* slots must not be NULL */
    serverAssert(slots != NULL);
    dictIterator *di = dictGetSafeIterator(src);
    dictEntry *de;
    while ((de = dictNext(di)) != NULL) {
        robj *key = dictGetKey(de);
        /* Check if key belongs to the slot range. */
        if (!slotRangeArrayContains(slots, keyHashSlot(key->ptr, sdslen(key->ptr))))
            continue;
        if (dictAddRaw(dst, key, NULL)) {
            incrRefCount(key);
        }
        dictDelete(src, key);
    }
    dictReleaseIterator(di);
}
```

## Glossary

- **Stream (Redis Streams)** — a Redis data type used to append and consume a log of events, similar to a message queue.
- **IDMP keys (Idempotent stream keys)** — special keys attached to entries in a stream, used to make sure that if the same entry gets written more than once (e.g. due to a retry), it doesn't get processed or stored twice. Redis checks these keys before adding new entries to detect and skip duplicates.
- **dict (dictionary)** — Redis's internal hash table implementation, used to store key-value pairs efficiently. Think of it as the C-level engine behind Redis keys.
- **Slot (hash slot)** — in a Redis Cluster, keys are distributed across 16384 "slots." Each node owns a range of slots, which is how data gets partitioned across the cluster.
- **Atomic Slot Migration (ASM)** — the process of moving one or more slots (and the keys in them) from one cluster node to another, done in a way that avoids data loss or inconsistency mid-move.
- **bgtrim (background trim)** — a background process that cleans up or removes expired/stale entries (in this case, expired IDMP stream keys) without blocking normal operations.