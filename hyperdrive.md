# Hyperdrive

Hyperdrive is the peer-to-peer data distribution protocol that powers Dat. It consists of two parts. First there is hypercore which is the core protocol and swarm that handles distributing append-only logs of any binary data. The second part is hyperdrive which adds a filesystem specific protocol on top of hypercore.

Throughout this document we'll use following tree terminology:

* `parent` - a node that has two children (odd numbered)
* `leaf` - a node with no children (even numbered)
* `sibling` - the other node with whom a node has a mutual parent
* `uncle` - a parent's sibling

## Hypercore

The goal of hypercore is to distribute append-only logs across a network of peers. Peers download parts of the logs from other peers and can choose to only download the parts of a log they care about. Logs can contain arbitrary binary data payloads.

## How Hypercore works

### Flat Trees

A flat tree is a simple way represent a binary tree as a list, associating every node of a binary tree with a numeric index. This simplifies the wire protocol for distributed applications that use tree structures. Flat trees are described in [PPSP RFC 7574 as "Bin numbers"](https://datatracker.ietf.org/doc/rfc7574/?include_text=1) and has been implemented in node (see the [flat-tree](https://github.com/mafintosh/flat-tree) module.)

A sample flat tree spanning 4 blocks of data looks like this:

```
0
  1
2  
    3
4
  5
6
```

1 is the parent of (0, 2), 5 is the parent of (4, 6), and 3 is the parent of (1, 5).

The even numbered entries represent data blocks (leaf nodes) and odd numbered entries represent parent nodes that have exactly two children.

The depth of a tree node can be calculated by counting the number of trailing 1s a node has in binary notation.

```
5 in binary = 101 (one trailing 1)
3 in binary = 011 (two trailing 1s)
4 in binary = 100 (zero trailing 1s)
```

If the number of leaf nodes is a power of 2, the flat tree will only have a single root. Otherwise it'll have more than one. Here is an example tree with six leafs and two roots (3 and 9).

```
0
  1
2
    3
4
  5
6

8
  9
10
```


## Merkle Trees

Merkle trees are useful for ensuring the integrity of content. A merkle tree is a binary tree where every leaf is a hash of a data block and every parent is the hash of both of its children.

Let's look at an example. Assume we have 4 data blocks, `(a, b, c, d)` and let `h(x)` be a hash function (the hyperdrive stack uses sha256 per default).

Using flat-tree notation, the merkle tree spanning these data blocks looks like this:

```
0 = h(a)
  1 = h(0 + 2)
2 = h(b)
    3 = h(1 + 5)
4 = h(c)
  5 = h(4 + 6)
6 = h(d)
```

An interesting property of merkle trees is that node 3 represents the entire data set using a cumulative hash. Therefore we only need to trust node 3 to verify all data in this example.


What happens when we expand our data set to contain 6 items `(a, b, c, d, e, f)`, so it has two roots?

```
0 = h(a)
  1 = h(0 + 2)
2 = h(b)
    3 = h(1 + 5)
4 = h(c)
  5 = h(4 + 6)
6 = h(d)

8 = h(e)
  9 = h(8 + 10)
10 = h(f)
```

As we learned above, there will only be a single root if the number of data blocks is an exact power of 2. So to make sure we can still reference all data with a single root, we hash all the roots together. In addition to hashing the roots we'll also include a bin endian uint64 binary representation of the corresponding node index. At most there will be `log2(number of data blocks)`.

Using the two above examples the final hashes would be:

```
hash1 = h(uint64be(#3) + 3)
hash2 = h(uint64be(#9) + 9 + uint64be(#3) + 3)
```

Each of these hashes can be used to fully verify each of the trees.

Let's look at another example. Assume we trust `hash1` and another person wants to send block `0` to us. To verify block `0` the other person would also have to send the sibling hash and uncles until it reaches a root and the other missing root hashes. For the first tree that would mean hashes `(2, 5)`.

Using these hashes we can reproduce `hash1` in the following way:

```
0 = h(block received)
  1 = h(0 + 2)
2 = (hash received)
    3 = h(1 + 5)
  5 = (hash received)
```

If `h(uint64be(#3) + 3) == hash1` then we know that data we received from the other person is correct. They sent us `a` and the corresponding hashes.

Since we only need the uncle hashes to verify the block, the amount of hashes we need is at worst `log2(number-of-blocks)` and the roots of the merkle trees which have the same complexity.

A merkle tree generator is available on npm through the [merkle-tree-stream](https://github.com/mafintosh/merkle-tree-stream) module.

## Merkle Tree Deduplication

Merkle trees have another great property. They make it easy to deduplicate content that is similar.

Assume we have two similar datasets:

```
(a, b, c, d, e)
(a, b, c, d, f)
```

These two datasets are the same except their last element is different. When generating merkle trees for the two data sets you'd get two different root hashes out.

However if we look a the flat-tree notation for the two trees:

```
0
  1
2
    3
4
  5
6

8
```

We'll notice that the hash stored at 3 will be the same for both trees since the first four blocks are the same. Since we also send uncle hashes when sending a block of data, we'll receive the hash for 3 when we request any block. If we maintain a simple index that maps a hash into the range of data it covers we can detect that we already have the data spanning 3 and we won't need to re-download that from another person.

```
1 -> (a, b)
3 -> (a, b, c, d)
5 -> (c, d)
```

This means that two datasets share a similar sequence of data the merkle tree helps you detect that.

## Signed Merkle Trees

As described above the top hash of a merkle tree is the hash of all its content. This has both advantages and disadvantages. An advantage is that you can always reproduce a merkle tree simply by having the data contents of a merkle tree. However, every time you add content to your data set, your merkle tree hash changes and you'll need to re-distribute the new hash.

Using a bit of cryptography we can make our merkle tree appendable. First, generate a cryptographic key pair that can be used to sign data using [ed25519](https://ed25519.cr.yp.to/) keys, as they are compact in size (32 byte public keys). A key pair (public key, secret key) can be used to sign data. Signing data means that if you trust a public key and you receive data and a signature for that data you can verify that a signature was generated with the corresponding secret key.

Instead of distributing the hash of a merkle tree, we instead distribute our public key. This allows us to then use our secret key to sign the merkle trees of our data set every time we append new data to it.

Assume we have a dataset with only a single item in it `(a)` and a key pair `(secret, public)`:

```
(a)
```

We generate a merkle tree for this dataset which will have the root `0` and sign the hash of the root (see the merkle tree section) with our secret key. If we want to send `a` to another person (and they trust our public key) we simply send `a` and the uncles needed to generate the roots plus our signature.

All new signatures verify the entire dataset since they all sign a merkle tree that spans all data. This serves two purposes. It makes sure that the dataset publisher cannot change old data. It also ensures that the publisher cannot share different versions of the same dataset to different persons without the other people noticing it (at some point they'll get a signature for the same node index that has different hashes if they talk to multiple people).

This technique has the added benefit that you can always convert a signed merkle tree to a normal unsigned one if you wish (or turn an unsigned tree into a signed tree).

In general you should send as wide as possible signed tree back when using signed merkle trees. This means that the the other person needs to verify fewer signatures, which increases performance for some platforms. It will also make it more efficient to detect if a tree has duplicated content.

## Block Tree Digest

When asking for a block of data we want to reduce the amount of duplicate hashes that are sent back.

In the merkle tree example for from earlier we ended up sending two hashes `(2, 5)` to verify block `0`.

```
// If we trust 3 then 2 and 5 are needed to verify 0

0
  1
2
    3
4
  5
6
```

Now if we ask for block `1` afterwards (`2` in flat tree notation) the other person doesn't need to send us any new hashes since we already received the hash for `2` when fetching block `0`.

If we only use non-signed merkle trees, the other person can easily calculate which hashes we already have after we tell them which blocks we've got.

This however isn't always possible if we use a signed merkle tree since the roots are changing. In general it also useful to be able to communicate that you have some hashes already without disclosing all the blocks you have. To communicate which hashes we have, we need to send two things: which uncles we have and whether or not we have any parent node that can verify the tree. Looking at the above tree that means if we want to fetch block `0` we need to communicate whether of not we already have the uncles `(2, 5)` and the parent `3`.

This information can be compressed into very small bit vector using the following scheme. Let the trailing bit donate whether or not the leading bit is a parent and not a uncle. Let the previous trailing bits denote wheather or not we have the next uncle. For example for block `0` the following bit vector `1011` is decoded the following way:

```
// for block 0

101(1) <-- tell us that the last bit is a parent and not an uncle
10(1)1 <-- we already have the first uncle, 2 so don't send us that
1(0)11 <-- we don't have the next uncle, 5
(1)000 <-- the final bit so this is parent. we have the next parent, 3
```

So using this digest the person can easily figure out that they only need to send us one hash, `5`, for us to verify block `0`.

The bit vector `1` (only contains a single one) means that we already have all the hashes we need so just send us the block.

These digests are very compact in size, only `(log2(number-of-blocks) + 2) / 8` bytes needed in the worst case. For example, if you are sharing one trillion blocks of data the digest would be `(log2(1000000000000) + 2) / 8 ~= 6` bytes long.
