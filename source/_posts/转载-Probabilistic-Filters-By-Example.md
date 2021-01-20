---
title: '[转载]Probabilistic Filters By Example'
date: 2021-01-20 06:58:28
tags: ['Algorithm']
categories:
cover: https://68.media.tumblr.com/953cf79fb096626b9d7b931f1fe605b1/tumblr_inline_oh2b4bgQtZ1qcg73w_540.png
---
<meta name="referrer" content="no-referrer" />

# [转载]Probabilistic Filters By Example

Probablistic filters are high-speed, space-efficient data structures that support set-membership tests with a one-sided error. These filters can claim that a given entry is *definitely not* represented in a set of entries, or *might* be represented in the set. That is, negative responses are conclusive, whereas positive responses incur a small false positive probability (FPP).

The trade-off for this one-sided error is space-efficiency. Cuckoo Filters and Bloom Filters require approximately 7 bits per entry at 3% FPP, regardless of the size of the entries. This makes them useful for applictations where the volume of original data makes traditional storage impractical.

**Bloom filters** have been in use since the 1970s and are well understood. Implementations are widely available. Variants exist that support deletion and counting, though with expanded storage requirements.

**Cuckoo filters** were described in [Cuckoo Filter: Practically Better Than Bloom](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf), a paper by researchers at CMU in 2014. Cuckoo filters improve on Bloom filters by supporting deletion, limited counting, and bounded FPP with similar storage efficiency as a standard Bloom filter.

## How do they work?

**Cuckoo Filters** operate by hashing an entry with one hash function, and inserting a small f-bit fingerprint of the entry into an open position in either of two alternate buckets. When both buckets are full, the filter recursively [kicks](https://en.wikipedia.org/wiki/Cuckoo_hashing) existing entries to their alternate buckets until space is found or attempts are exhausted. Lookups repeat the hash function and check both buckets for the fingerprint. When a matching fingerprint is not found, the entry is *definitely not* in the filter. When a matching fingerprint is found in either bucket, the entry *might* be in the filter. False positives occur when another entry inserted a mathcing fingerprint into either of the two checked buckets. Deletion is supported by removing one instance of an entry's fingerprint from either bucket. Counting is supported by inserting multiple fingerprints of the same value into the same pair of buckets.

**Bloom Filters** operate by hashing an entry with k hash functions, and setting k bits within a bit vector upon insertion. Lookups repeat the k hash functions and check the corresponding bits. When any checked bit is not set, the entry is *definitely not* in the filter. When all checked bits are set, the entry *might* be in the filter. False positives occur when all checked bits happen to be set by any combination of previously inserted entries.

## What are they good for?

Probabilistic filters are used in a variety of applications where slow or expensive operations can be avoided prior to execution by a consulting comparitavely fast or cheap set membership test.

**DB query optimization:** data stored in a database can be inserted into a probabilistic filter. Later, prior to querying for the data, the filter can be consulted to test whether the data exists. When the filter response is negative, an unnecessary DB query can be avoided.

**Edge filtering:** filter data can be distributed to the edges of networks, similar to edge caching, where queries for potentially non-existent data received at the edge can be filtered out quickly. Unlike a cache, a copy of original data is not stored in the filter.

## Which should I choose?

Choose Cuckoo, if available, unless your application is timing sensitive on insertion. Though Cuckoo filters outperform Bloom filters on insertion at first - O(1) vs O(k), respectively - their insertion performance drops off due to recursive entry "kicking" as the filter approaches its max capacity.

> ...for reasonably large sized sets, for the same false positive rate as a corresponding Bloom filter, cuckoo filters use less space than Bloom filters, are faster on lookups (but slower on insertions/to construct), and amazingly also allow deletions of keys (which Bloom filters cannot do). -[Michael Mitzenmacher (2014)](http://mybiasedcoin.blogspot.com/2014/10/cuckoo-filters.html)

|                   | Cuckoo Filter                                                | Standard Bloom Filter                               | Counting Bloom Filter                                        |
| ----------------- | ------------------------------------------------------------ | --------------------------------------------------- | ------------------------------------------------------------ |
| Insert            | Variable. O(1) amortized longer as load factor approaches capacity | Fixed. O(k)                                         | Fixed. O(k)                                                  |
| As load increases | FPP trends toward desired maxInsertions *may* be rejected if counting or deletion support is enabled | FPP trends toward 100%Insertions cannot be rejected | FPP trends toward 100%Insertions *may* be rejected           |
| Lookup            | O(1) Maximum of two buckets to check                         | O(k)                                                | O(k)                                                         |
| Count             | O(1) minimal suport: max == entries per bucket X 2           | *unsupported*                                       | O(k)                                                         |
| Delete            | O(1) Maximum of two buckets to inspect                       | *unsupported*                                       | O(k)                                                         |
| Bits per entry    | smaller when desired FPP <= 3%                               | smaller when desired FPP > 3%                       | larger than Cuckoo & Standard Bloom multiplied by number of bits per counter |
| Bits per entry    | 1.05 [ log2(1/FPP) + log2(2b) ] best when FPP <= 0.5% "semi-sort cuckoo" best when FPP <= 3% | 1.44 log2(1/FPP) best when FPP > 0.5%               | c [ 1.44 log2(1/FPP) ] where c is the number of bits per counter, e.g. 4 |
| Availability      | limited (as of early 2016) [cpp](https://github.com/efficient/cuckoofilter) [java](https://github.com/bdupras/guava-probably) [go](https://github.com/seiflotfy/cuckoofilter) | widely available                                    | widely available                                             |

## See also

- [Cuckoo Filter: Practically Better Than Bloom](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf), Bin Fan, et al.
- [Cuckoo Filters](http://mybiasedcoin.blogspot.com/2014/10/cuckoo-filters.html), Michael Mitzenmacher
- [Bloom Filters by Example](http://billmill.org/bloomfilter-tutorial/), Bill Mill
- [Bloom filter on Wikipedia](https://en.wikipedia.org/wiki/Bloom_filter)