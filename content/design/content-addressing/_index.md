+++
title = "Content Addressing"
description = ""
template="design/page.html"
[extra]
section="2."
+++

## Content Addressed Blobs

Iroh works with _blobs_ of opaque data, which are often the bytes of a file. Blobs can be of arbitrary size ranging from 0-u64MAX bytes.

All blobs within Iroh are referred to by the BLAKE3 [1] hash of contents. BLAKE3 is a _tree hashing_ algorithm, that splits its input into uniform chunks and arranges them as the leaves of a binary tree, producing intermediate _chunk hashes_ that accumulate up to a _root hash._ Iroh uses the 32 byte root hash (or just “hash”) as an immutable blob identifier. 

<img class="figure" src="/design/content-addressing/fig_1_blob.svg" />

We leverage the tree hash structure of BLAKE3 as a basis for incremental verification when data is sent over the network, as described by Section 6.4 “Verified Streaming” in [1] and implemented in [2]. Iroh caches all chunk hashes as external metadata, leaving the unaltered input blob as the canonical source of bytes. Verified streaming also facilitates _range requests:_ fetching a verifiable contiguous subsequence of a blob by streaming only the portions of the BLAKE3 binary tree required to verify the designated subsequence.

Chunk hashes are distinct from root hashes, and only used during data transfer. The chunk size of BLAKE3 is a tunable constant that defaults to 1KiB, which results in a 6% overhead on on the size of the input blob. Increasing the chunk size reduces overhead, at the cost of requiring more data transfer before an incremental verification checkpoint is reached. The chunk size constant can be modified & recalculated *without affecting the root hash.* This opens the door to experiment with different chunk size constants, even at runtime. We intend to investigate chunk size optimization in future work.

Root hashes are expressed as a Content identifier (CID) as defined in [3], making Iroh an *IPFS system* capable of interoperating with other systems that use CIDs. In contrast to other IPFS systems, only root hashes are valid content identifiers, which enforces a strict 1-1 relationship between a Content Identifier and blob. This 1-1 relationship brings Iroh into alignment with common whole-file checksum systems. A naive implementation of Iroh can skip verified streaming entirely and use the the CID as a whole-file checksum.

## Collections

A *Collection* is an ordered set of named *links* to blobs. Collection link counts can range from 0-billions. Collections are the only means of relating blobs within iroh, and form the basis of synchronization & querying. Collections are true sets, and cannot be nested to form graphs. Internally collections are structured as a _radix tree_ [5] to support efficient prefix querying.

A link is a triple of `name, CID`. *Name* is an opaque, arbitrary sequence of bytes that labels the link. These are typically UTF-8 strings, but are always treated and sorted as bytes. Names must be unique across the collection, and order the set. *CID* is the content identifier for the linked blob.

Collections have an optional _header_ section that stores the number of items in the collection and offsets to items within the list, forming a skip list index.

<img class="figure" src="/design/content-addressing/fig_2_collection.svg" />

Collections are content addressed in the exact same manner as blobs, but use differing CID multicodec identifier to distinguish them from opaque blobs. Like blobs, collections can be seeked into using byte offsets.

## Collection Querying

Queries can be performed to animate collection sets into graph structures. A typical example is constructing file system directories from the set using slash separators combined with a prefix query. We have yet to begin researching collection querying and will have more to report in the future, but know that prefix querying will be supported at a minimum.




<a class="next-page-button" href="/design/dsht">
Next: 3. Distributed Sloppy Hash Table
</a>

# References

1. **blake3**<br />
[https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf)
2. **bao**<br />
[https://github.com/oconnor663/bao](https://github.com/oconnor663/bao)
3. **CID (Content IDentifier): Self-describing content-addressed identifiers for distributed systems**<br />
[https://github.com/multiformats/cid](https://github.com/multiformats/cid)
4. **multihashes**<br />
[https://multiformats.io/multihash/](https://multiformats.io/multihash/)
5. **Radix Tree**<br />
[https://en.wikipedia.org/wiki/Radix_tree](https://en.wikipedia.org/wiki/Radix_tree)