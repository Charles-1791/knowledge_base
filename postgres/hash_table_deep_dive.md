# Postgres Source Code Walk-through : HTAB, a Generic Hash Table

The following is the source code of a hash table, written in `dynahash.c`

```c
struct HTAB
{
	HASHHDR    *hctl;			/* => shared control information */
	HASHSEGMENT *dir;			/* directory of segment starts */
	HashValueFunc hash;			/* hash function */
	HashCompareFunc match;		/* key comparison function */
	HashCopyFunc keycopy;		/* key copying function */
	HashAllocFunc alloc;		/* memory allocator */
	MemoryContext hcxt;			/* memory context if default allocator used */
	char	   *tabname;		/* table name (for error messages) */
	bool		isshared;		/* true if table is in shared memory */

	/* freezing a shared table isn't allowed, so we can keep state here */
	bool		frozen;			/* true = no more inserts allowed */

	/* We keep local copies of these fixed values to reduce contention */
	Size		keysize;		/* hash key length in bytes */
	int64		ssize;			/* segment size --- must be power of 2 */
	int			sshift;			/* segment shift = log2(ssize) */

	/*
	 * In a USE_VALGRIND build, non-shared hashtables keep an slist chain of
	 * all the element blocks they have allocated.  This pacifies Valgrind,
	 * which would otherwise often claim that the element blocks are "possibly
	 * lost" for lack of any non-interior pointers to their starts.
	 */
#ifdef USE_VALGRIND
	slist_head	element_blocks;
#endif
};
```

Since there are so many fields, explaining them one by one can be dreary, I decide to talk about them when used. Now, go through the initialization process first.

## Initialization

An `HTAB` is created by calling:

```c
HTAB *
hash_create(const char *tabname, int64 nelem, const HASHCTL *info, int flags)
```

where:

- `tabname` is the name of the hash table, used only for debugging.
- `nelem` specifies the estimated maximum number of elements.
- `info` provides configuration details such as the hash function and match function.
- `flags` is a bitmask specifying which fields in `info` are valid.

The function begins by allocating memory for the `HTAB` structure, along with space to store the table name:

```c
/* Initialize the hash header, plus a copy of the table name */
	hashp = (HTAB *) MemoryContextAlloc(CurrentDynaHashCxt,
										sizeof(HTAB) + strlen(tabname) + 1);
	MemSet(hashp, 0, sizeof(HTAB));

	hashp->tabname = (char *) (hashp + 1);
	strcpy(hashp->tabname, tabname);
```

Here, memory is allocated from a specific memory context (which you can think of as a memory pool). Notice that the allocation size is `sizeof(HTAB) + strlen(tabname) + 1`. The extra `strlen(tabname) + 1` bytes are reserved to store the table’s name. This is confirmed by the assignment `hashp->tabname = (char *)(hashp + 1)`, which points `tabname` to the memory immediately following the `HTAB` structure.

```c
/*
	 * Select the appropriate hash function (see comments at head of file).
	 */
	if (flags & HASH_FUNCTION)
	{
		Assert(!(flags & (HASH_BLOBS | HASH_STRINGS)));
		hashp->hash = info->hash;
	}
	else if (flags & HASH_BLOBS)
	{
		Assert(!(flags & HASH_STRINGS));
		/* We can optimize hashing for common key sizes */
		if (info->keysize == sizeof(uint32))
			hashp->hash = uint32_hash;
		else
			hashp->hash = tag_hash;
	}
	else
	{
		/*
		 * string_hash used to be considered the default hash method, and in a
		 * non-assert build it effectively still is.  But we now consider it
		 * an assertion error to not say HASH_STRINGS explicitly.  To help
		 * catch mistaken usage of HASH_STRINGS, we also insist on a
		 * reasonably long string length: if the keysize is only 4 or 8 bytes,
		 * it's almost certainly an integer or pointer not a string.
		 */
		Assert(flags & HASH_STRINGS);
		Assert(info->keysize > 8);

		hashp->hash = string_hash;
	}

```

Then, we need to assign a hash function to the table. If there is a given function in `info`, we directly use it; if `HASH_BLOBS` is set, we assign corresponding hash function based on the size of the key; otherwise, `string_hash` is used by default.

```c
	/*
	 * If you don't specify a match function, it defaults to string_compare if
	 * you used string_hash, and to memcmp otherwise.
	 *
	 * Note: explicitly specifying string_hash is deprecated, because this
	 * might not work for callers in loadable modules on some platforms due to
	 * referencing a trampoline instead of the string_hash function proper.
	 * Specify HASH_STRINGS instead.
	 */
	if (flags & HASH_COMPARE)
		hashp->match = info->match;
	else if (hashp->hash == string_hash)
		hashp->match = (HashCompareFunc) string_compare;
	else
		hashp->match = memcmp;
```

Similarly, `match` function must also be determined. It's a comparator for user defined key.

```c
	/*
	 * Similarly, the key-copying function defaults to strlcpy or memcpy.
	 */
	if (flags & HASH_KEYCOPY)
		hashp->keycopy = info->keycopy;
	else if (hashp->hash == string_hash)
	{
		/*
		 * The signature of keycopy is meant for memcpy(), which returns
		 * void*, but strlcpy() returns size_t.  Since we never use the return
		 * value of keycopy, and size_t is pretty much always the same size as
		 * void *, this should be safe.  The extra cast in the middle is to
		 * avoid warnings from -Wcast-function-type.
		 */
		hashp->keycopy = (HashCopyFunc) (pg_funcptr_t) strlcpy;
	}
	else
		hashp->keycopy = memcpy;
```

Next, the method to copy a key is defined.

We now comes to the initialization of `HASHHDR` - the runtime metadata of the hash table. In order to understand it better, we need to know how does a hash table store elements, namely, its memory layout.

## Memory Layout

### C-style structure embedding

Suppose we want to implement a linked list that can store any user-defined type. In C++, we’d naturally write something like:

```c++
template <typename T>
struct ListNode {
	ListNode* next;
  T* element;
}
```

In C, however, we don’t have templates. Instead, we achieve a similar effect by defining a small “header” structure that user-defined nodes embed at the **beginning** of their own struct:

```c
struct ListHeader {
	ListHeader* next;
}
struct CustomizedNode {
  ListHeader header_part;
  // extra part
}
```

This clever design ensures that for any user-defined node, the first few bytes always form a valid header. In other word, since the header is at offset 0, the list implementation can safely **reinterpret** any user node as a `ListHeader` and manipulate it without knowing the rest of the struct.

Conceptually, this trick mimics a simple form of inheritance in C: the header acts like a *base class*, and the user-defined node behaves like a *derived class* that inherits from it.

Postgres' hash table is implemented in this way: each hash bucket is a linked list, and each entry begins with a header:

```c
typedef struct HASHELEMENT
{
	struct HASHELEMENT *link;	/* link to next entry in same bucket */
	uint32		hashvalue;		/* hash function result for this entry */
} HASHELEMENT;

```

Postgres adds one more constraint: the bytes immediately following the header must store the key. As a result, a user-defined hash table entry has the following memory layout:

```
+---------------------+
| HASHELEMENT header  |   <- PostgreSQL internal header
+---------------------+
| user key bytes      |   <- this has length = keysize
+---------------------+
| user data bytes     |   <- whatever your struct stores
+---------------------+
```

Since the keys always follow `HASHELEMENT`, Postgres can compute the key’s starting address with a simple macro:

```c
/*
 * Key (also entry) part of a HASHELEMENT
 */
#define ELEMENTKEY(helem)  (((char *)(helem)) + MAXALIGN(sizeof(HASHELEMENT)))
```

### Segment-Bucket Layout

Postgres’ hash table stores its buckets indirectly through an array of *segments*, referenced by the `dir` pointer:

```c
HASHSEGMENT *dir;			/* directory of segment starts */
```

Each segment is simply an array of bucket pointers:

```c
// dynahash.c
/* A hash bucket is a linked list of HASHELEMENTs */
typedef HASHELEMENT *HASHBUCKET;
/* A hash segment is an array of bucket headers */
typedef HASHBUCKET *HASHSEGMENT;
```

A `HASHBUCKET` is the head pointer of a linked list of `HASHELEMENT`s:

```c
// hsearch.h
typedef struct HASHELEMENT
{
	struct HASHELEMENT *link;	/* link to next entry in same bucket */
	uint32		hashvalue;		/* hash function result for this entry */
} HASHELEMENT;
```

As discussed earlier in [C styled structure embedding](###C styled structure embedding) , `HASHELEMENT` serves like the **base class** for every stored entry, with user data following immediately behind it in memory.

Visually, the structure looks like this:

```c
  HTAB
+-------+       Segments
|  dir  |--> +---------------+                                      Buckets  
+-------+    |     seg 0     |-------------------------------->   +----------+
             +---------------+             Buckets                | bucket 0 |---> list
             |     seg 1     |-------->  +----------+             +----------+
             +---------------+           | bucket 4 |---> list    | bucket 1 |---> list
                                         +----------+             +----------+
                                         | bucket 5 |---> list    | bucket 2 |---> list
                                         +----------+             +----------+
                                         | bucket 6 |---> list    | bucket 3 |---> list
                                         +----------+             +----------+
                                         | reserved |---> NULL
                                         +----------+
            
```

During the initialization, Postgres first allocates the segment directory, and then allocates each segment (each being an array of buckets):

```c
static bool
init_htab(HTAB *hashp, int64 nelem)
{
  ...
	/* Allocate a directory */
	if (!(hashp->dir))
	{
		CurrentDynaHashCxt = hashp->hcxt;
		hashp->dir = (HASHSEGMENT *)
			hashp->alloc(hctl->dsize * sizeof(HASHSEGMENT));
		if (!hashp->dir)
			return false;
	}
  /* Allocate initial segments */
	for (segp = hashp->dir; hctl->nsegs < nsegs; hctl->nsegs++, segp++)
	{
		*segp = seg_alloc(hashp);
		if (*segp == NULL)
			return false;
	}
  ...
}
```

Like the notion of `capacity` and `size` of `std::vector` in `c++`, hash table allocates `dsize` segments in one sitting, with `nsegs` recording how many of them are actually used. 

``` c
struct HASHHDR
{
  ...
  int64		dsize;			/* directory size */
  int64		nsegs;			/* number of allocated segments (<= dsize) */
  ...
}
```

Given the hash value of a key, a bucket id is computed. Postgres takes the id's higher bits as segment index, and the next `sshift` lower bits as bucket offset within a segment. Each segment contains exactly `ssize = 2^sshift` buckets. Importantly, both `ssize` and `sshift` are fixed at table creation and never change:

```c
struct HASHHDR
{
  ...
  int64		ssize;			/* segment size --- must be power of 2 */
  int			sshift;			/* segment shift = log2(ssize) */
  ...
}
```

In comparison, `dsize` is mutable and will be doubled when table expands, see [table expansion process](###table expansion process) for details.

## Entry Look Up

The lookup procedure begins with `hash_search`, which internally calls `hash_search_with_hash_value`:

````c
void *
hash_search(HTAB *hashp,
			const void *keyPtr,
			HASHACTION action,
			bool *foundPtr)
{
	return hash_search_with_hash_value(hashp,
									   keyPtr,
									   hashp->hash(keyPtr, hashp->keysize),
									   action,
									   foundPtr);
}
````

The `action` parameter controls what to do after the lookup: search only, insert if missing, delete, etc.

**Load Check and Expansion Trigger**

If the action is insertion, hash table checks the current load by comparing `nentries` and ` max_bucket`. If the former is larger, expansion is triggered. (see  [table expansion process](###table expansion process)).

```c
	/*
	 * If inserting, check if it is time to split a bucket.
	 *
	 * NOTE: failure to expand table is not a fatal error, it just means we
	 * have to run at higher fill factor than we wanted.  However, if we're
	 * using the palloc allocator then it will throw error anyway on
	 * out-of-memory, so we must do this before modifying the table.
	 */
	if (action == HASH_ENTER || action == HASH_ENTER_NULL)
	{
		/*
		 * Can't split if running in partitioned mode, nor if frozen, nor if
		 * table is the subject of any active hash_seq_search scans.
		 */
		if (hctl->freeList[0].nentries > (int64) hctl->max_bucket &&
			!IS_PARTITIONED(hctl) && !hashp->frozen &&
			!has_seq_scans(hashp))
			(void) expand_table(hashp);
	}
```

**Determining the Bucket**

After the expansion check, the hash value is passed to `hash_initial_lookup`, which returns the bucket ID and the address of the target bucket (a pointer to a `HASHBUCKET` pointer).

```c
/*
 * Do initial lookup of a bucket for the given hash value, retrieving its
 * bucket number and its hash bucket.
 */
static inline uint32
hash_initial_lookup(HTAB *hashp, uint32 hashvalue, HASHBUCKET **bucketptr)
{
	HASHHDR    *hctl = hashp->hctl;
	HASHSEGMENT segp;
	int64		segment_num;
	int64		segment_ndx;
	uint32		bucket;

	bucket = calc_bucket(hctl, hashvalue);

	segment_num = bucket >> hashp->sshift;
	segment_ndx = MOD(bucket, hashp->ssize);

	segp = hashp->dir[segment_num];

	if (segp == NULL)
		hash_corrupted(hashp);

	*bucketptr = &segp[segment_ndx];
	return bucket;
}

```

Here, `calc_bucket` determines the bucket ID, whose upper bits select the segment (`segment_num`), while the lower `sshift` bits select the bucket within that segment (`segment_ndx`).

**Scanning the Bucket**

Once we obtain the bucket pointer, we traverse its linked list of `HASHELEMENT`s to locate a matching entry:

```c
while (currBucket != NULL)
	{
		if (currBucket->hashvalue == hashvalue &&
			match(ELEMENTKEY(currBucket), keyPtr, keysize) == 0)
			break;
		prevBucketPtr = &(currBucket->link);
		currBucket = *prevBucketPtr;
#ifdef HASH_STATISTICS
		hctl->collisions++;
#endif
	}
if (foundPtr)
		*foundPtr = (bool) (currBucket != NULL);

```

The lookup first compares the stored `hashvalue` (cheap check) and then calls `match` to compare the user-provided key data. If no match is found, `currBucket` becomes `NULL`, and the caller may proceed with insertion or deletion depending on the original action.

## Table Expansion And Linear Hashing

### Mapping a hash value to a bucket id

In the previous chapter, we explained how to locate an entry once its bucket ID is known: use the higher bits as the segment index and the lower bits as the offset within that segment. What we haven't addressed is **how a hash value is mapped to a bucket ID**.

A conventional hash table computes:

```
bucket_id = hash_value % num_bucket
```

But this approach is not rehashing-friendly - when the table grows, rehashing is needed and `num_bucket` gets increment. Every  entry's `bucket_id` needs to be recomputed, which introduce a great performance penalty. 

By comparison, `Postgres` employs linear hashing, minimizing redistribution( see [data redistribution](###data redistribution) for details). The table maintains three members:

```c
struct HASHHDR {
	...
	uint32		max_bucket;		/* ID of maximum bucket in use */
	uint32		high_mask;		/* mask to modulo into entire table */
	uint32		low_mask;		/* mask to modulo into lower half of table */
	...
}
```

- `max_bucket` is simply the index of the last active bucket, i.e. `number_of_buckets - 1`. 
- `high_mask` is an all-one bitmask with the same number of bits as `max_bucket`. For instance, assume `max_bucket` is 43,  `101011` in binary form, then `high_mask` is 63, `111111` in binary. Mathematically, we have `high_mask = next_power_of_two(max_bucket) - 1`.
-  `low_mask`  is just `high_mask` right shift 1: `lower_mask = high_mask >> 1`.

To determine the bucket for a given `hash_val`, PostgreSQL first tries:

```c
bucket = hash_val & high_mask
```

If the result is larger than `max_bucket`, it means this bucket hasn’t been allocated yet, so it recomputes using fewer bits:

```c
bucket = hash_val & low_mask
```

This looks like:

```
           	     1001001   <- max bucket
                 1111111   <- high mask
010110101011101001010110   <- hash value
                  111111   <- low mask
```

And here is the corresponding code:

```c
/* Convert a hash value to a bucket number */
static inline uint32
calc_bucket(HASHHDR *hctl, uint32 hash_val)
{
	uint32		bucket;

	bucket = hash_val & hctl->high_mask;
	if (bucket > hctl->max_bucket)
		bucket = bucket & hctl->low_mask;

	return bucket;
}
```

### table expansion process

Recall in [Entry Look Up](# Entry Look Up), the function `expand` is invoked when `hctl->freeList[0].nentries > (int64) hctl->max_bucket`, which can be roughly interpreted as the load factor reaches 1. This function adds a single hash bucket to the table:

```c
/*
 * Expand the table by adding one more hash bucket.
 */
static bool
expand_table(HTAB *hashp)
```

The new bucket’s ID is simply the current `max_bucket + 1`, so it is placed immediately after the last existing bucket:

```c
  HTAB
+-------+        Segments
|  dir  |--> +---------------+                                      Buckets  
+-------+    |     seg 0     |-------------------------------->   +----------+
             +---------------+             Buckets                | bucket 0 |---> list
             |     seg 1     |-------->  +----------+             +----------+
             +---------------+           | bucket 4 |---> list    | bucket 1 |---> list
                                         +----------+             +----------+
                                         | bucket 5 |---> list    | bucket 2 |---> list
                                         +----------+             +----------+
                                         | bucket 6 |---> list    | bucket 3 |---> list
                                         +----------+             +----------+
                                         | bucket 7 |---> list
                                         +----------+
            
```

If the current segment has space, the new bucket is appended at the end, just like placing bucket 1 after bucket 0. If the segment is full, the bucket moves to the next segment. For example, bucket 4 is the first bucket in segment 1 because segment 0 has no remaining slots.

```c
// in expand_table:
	new_bucket = hctl->max_bucket + 1;
	new_segnum = new_bucket >> hashp->sshift;
	new_segndx = MOD(new_bucket, hashp->ssize);

	if (new_segnum >= hctl->nsegs)
	{
		/* Allocate new segment if necessary -- could fail if dir full */
		if (new_segnum >= hctl->dsize)
			if (!dir_realloc(hashp))
				return false;
		if (!(hashp->dir[new_segnum] = seg_alloc(hashp)))
			return false;
		hctl->nsegs++;
	}

	/* OK, we created a new bucket */
	hctl->max_bucket++;

```

A natural question arises: **what if all current segments are used?**

For instance, adding bucket 8 would require segment 2, which might not exist yet. In such cases, additional segments are to be allocated to accommodate new bucket, and that's why `dir_realloc(hashp)` is called. 

PostgreSQL always doubles the number of segments during reallocation. This is because the segment ID is determined by the high-order bits of the bucket ID. Expanding the table effectively uses **one more high bit**, which doubles the segment address space.

```c
static bool
dir_realloc(HTAB *hashp)
{
	...
	/* Reallocate directory */
	new_dsize = hashp->hctl->dsize << 1;
	old_dirsize = hashp->hctl->dsize * sizeof(HASHSEGMENT);
	new_dirsize = new_dsize * sizeof(HASHSEGMENT);

	old_p = hashp->dir;
	CurrentDynaHashCxt = hashp->hcxt;
	p = (HASHSEGMENT *) hashp->alloc((Size) new_dirsize);

	if (p != NULL)
	{
		memcpy(p, old_p, old_dirsize);
		MemSet(((char *) p) + old_dirsize, 0, new_dirsize - old_dirsize);
		hashp->dir = p;
		hashp->hctl->dsize = new_dsize;

		/* XXX assume the allocator is palloc, so we know how to free */
		Assert(hashp->alloc == DynaHashAlloc);
		pfree(old_p);

		return true;
	}

	return false;
}
```

Here, `memcpy` copies all existing segment pointers into the newly allocated directory. The remaining space is zeroed out, preparing room for future segments.

### data redistribution

After a new bucket is added, some entries may map to a different bucket and therefore need to be moved. A key property of  linear hashing is that only one existing bucket is ever subject to redistribution when a new bucket is created. More precisely, when we add a new bucket with id `new_bucket`, we only need to inspect the bucket with `id = new_bucket & low_mask`. 

```c
	/*
	 * *Before* changing masks, find old bucket corresponding to same hash
	 * values; values in that bucket may need to be relocated to new bucket.
	 * Note that new_bucket is certainly larger than low_mask at this point,
	 * so we can skip the first step of the regular hash mask calc.
	 */
	old_bucket = (new_bucket & hctl->low_mask);
```

See [Proof: Only One Bucket Requires Rehashing](###Proof: Only One Bucket Requires Rehashing) for details.

One by one, we check entries in that bucket, and move those whose bucket id changes:

```c
	old_segnum = old_bucket >> hashp->sshift;
	old_segndx = MOD(old_bucket, hashp->ssize);

	old_seg = hashp->dir[old_segnum];
	new_seg = hashp->dir[new_segnum];

	oldlink = &old_seg[old_segndx];
	newlink = &new_seg[new_segndx];

	for (currElement = *oldlink;
		 currElement != NULL;
		 currElement = nextElement)
	{
		nextElement = currElement->link;
		if ((int64) calc_bucket(hctl, currElement->hashvalue) == old_bucket)
		{
			*oldlink = currElement;
			oldlink = &currElement->link;
		}
		else
		{
			*newlink = currElement;
			newlink = &currElement->link;
		}
	}
```

### Proof: Only One Bucket Requires Rehashing

Assume an entry with hash value `hash_val` is stored in bucket `old_bucket` before the new bucket is added. We want to determine under what conditions this entry needs to be moved.

When a new bucket is added, `max_bucket++`, and this leads to two possible scenarios:

- Case 1: `max_bucket` grows from `2^n - 1` to `2^n`. its bit length increases by 1, so  `high_mask` and `low_mask` are updated.

```c
static bool expand_table(HTAB *hashp)
{
	...
	hctl->max_bucket++;
	...
	/*
	 * If we crossed a power of 2, readjust masks.
	 */
	if ((uint32) new_bucket > hctl->high_mask)
	{
		hctl->low_mask = hctl->high_mask;
		hctl->high_mask = (uint32) new_bucket | hctl->low_mask;
	}
	...
}
```

- Case 2: Otherwise, we haven’t crossed a power-of-two boundary, so neither mask changes.

We now analyze these two cases separately, focusing on the algorithm of getting bucket_id from hash value.

```c
/* Convert a hash value to a bucket number */
static inline uint32
calc_bucket(HASHHDR *hctl, uint32 hash_val)
{
	uint32		bucket;

	bucket = hash_val & hctl->high_mask;
	if (bucket > hctl->max_bucket)
		bucket = bucket & hctl->low_mask;

	return bucket;
}
```

#### case 1

For case 1, let’s call the masks before the expansion (`max_bucket == 2^n − 1`) `old_low_mask` and `old_high_mask`, and the masks after the expansion (`max_bucket == 2^n`) `new_low_mask` and `new_high_mask`. By the update rule, we immediately have:

> old_high_mask == new_low_mask

Before the expansion, when `max_bucket` was still `2^n − 1`, we also had:

> old_high_mask == old_max_bucket

So:

> hash_val & old_high_mask <= old_max_bucket

This means the `if` condition in `calc_bucket()` never fired, and therefore

> old_bucket_id = hash_val & old_high_mask

When `max_bucket` becomes `2^n`, we want to determine when an entry’s new bucket ID differs from its old one.
- sub-case 1: the if-condition holds 

  Then:

  > new_bucket_id = hash_val & new_low_mask

  But `new_low_mask == old_high_mask`, so this is exactly:

  > new_bucket_id = old_bucket_id

  and the entry does not need to move.

- sub-case 2: the if-condition fails

  This means:

  > new_bucket_id = hash_val & new_high_mask

  and 

  > hash_val & new_high_mask <= new_max_bucket ... [1]

  Note that `new_max_bucket` is the smallest number with the same number of bits as `new_high_mask`, so the only way [1] holds  is when: 

  >  hash_val & new_high_mask == new_max_bucket ... [2]

  Since `new_max_bucket` has exactly one bit set — the new highest bit — masking the same hash value with the *shorter* `old_high_mask` necessarily yields:

  >  old_bucket_id = hash_val & old_high_mask == 0

  In addition, since the old bucket id is 0, it can also be represented by: 

  > old_bucket_id = hash_val & old_low_mask

  Here is an example:

```
								1111111   <- new high mask  
  100101001001001000000   <- hash value
                 111111   <- old high mask = new low mask
  So:
  new_bucket_id = 1000000
  old_bucket_id =  000000
```
Combining these two sub-cases, we can conclude that:

> any entries need to be moved must be in bucket 0, mathematically, its bucket id = new_max_bucket & old_low_mask

#### case 2

In case 2, neither `low_mask` nor `high_mask` changes. Looking at the `calc_bucket`, we can reason as follows: 

For an entry to move, its old bucket ID must differ from its new bucket ID. In other words, two calls to `calc_bucket()` - before and after adding the new bucket - must return different results. Since  `hash_value`, `high_mask`, and `low_mask` remain the same, there are only two ways this could happen:

- the `if` condition succeeds before the expansion and fails afterward, or
- the `if` condition fails before the expansion and succeeds afterward.

**Claim: the second situation is impossible**. 

Prove by contradiction: 

If the `if` condition fails in the earlier call, then:

>  hash_value & high_bitmask <= old_max_bucket ... [1]

If the `if` condition succeeds in the later call, then:

>  hash_value & high_bitmask > new_max_bucket ... [2]

And of course:

> new_max_bucket > old_max_bucket ... [3]

Combining [1] [2] and [3] yields:

>  hash_value & high_bitmask > new_max_bucket > old_max_bucket >= hash_value & high_bitmask

which is clearly a contradiction. 

**So this case cannot occur.**

Now consider the first possibility:

If the `if` condition succeeds in the earlier call, then:

> hash_value & high_bitmask > old_max_bucket ...  [1] 

If the `if` condition fails in the later call, then:

> hash_value & high_bitmask <= new_max_bucket ... [2] 

We further have:

> new_max_bucket == old_max_bucket + 1 ... [3] 

Combining [1] [2] and [3] yields:

> hash_value & high_bitmask == new_max_bucket ... [4]

From [4] we can derive:

> old_bucket_id 
>
> == hash_value & low_bitmask 
>
> == (hash_value & high_bitmask) & low_bitmask 
>
> == new_max_bucket & low_bitmask

Thus the only entries that need to be moved are those in bucket: `new_max_bucket & low_mask`

#### combine these two cases

Putting both cases together, we can draw a final conclusion:

> **Whenever a new bucket is added to the hash table, the only entries that may need to be relocated are those originally in bucket `new_max_bucket & low_mask`.**

This follows directly from the earlier analysis: regardless of whether `low_mask` changes (Case 1) or stays the same (Case 2), the condition for an entry's bucket ID to change is always: its original bucket must match `new_max_bucket & low_mask`.

