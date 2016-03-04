## RFC Hypercore appendable feeds

An appendable hypercore feed is a distributed single-writer append-only feed of data.
It is identified by a ECC public key that signs entry items (and merkle trees).

What to we mean by that?

If you are familiar with hypercore static feeds they work like this

```
              
      3
  1       5   
0   2   4   6   8 
```

The even numbers represent data blocks (small binary values (usually 16kb))
and the uneven numbers represent merkle tree nodes that are the hashes of their neighbors

The above static hypercore feed would be identified by the hash of 3, 8 that those tree nodes
span the entire data set.

Now this works great for sharing a set of files or similar static data but has limitations
if you want to share data is appendable. Everytime you append to the dataset you'd get a new hash
that you'd have to share and it you'd need to somehow distribute this hash to all existing clients.

If your dataset updates multiple times per day this quicky becomes infeasible

This is where appendable feeds come in.

Instead of identifying a feed by the hash we generate a key pair (secret, public) and identify the feed
using the public key. If we are using ECC keys, the public key is usually small (32 bytes) which makes it
appear similar to a hash as well

Every time you append to a feed you should use the secret key to sign the current merkle tree.

```
# first item

0 <-- just sign 0
```

```
# second item

  1    <-- sign 1 (spans 0 and 2)
0   2
```

```
# third item

  1
0   2    4 <-- sign hash(1,4)
```

```
#fourth item

       3 <-- sign 3
  1        5
0   2    4   6
```

When two peers (peerA and peerB) connect they first exchange a tree digest (usually a bitfield) that allows both peers
to know which tree nodes the other peer has.

When peerA requests a data block (an even number) that peerB has, one of two things will happen:

### Scenario A

peerB looks at peerA's tree digest and notices that peerA has a parent (or ancestor) tree node to the
data block it is requesting. peerB then just send the uncles hashes peerA is missing to verify the data block

(this is similar to how static feeds work)

```
peerB's digest

      3
  1       5
0   2   4   6

peerA's digest:

      3
      
peerA request 0, which makes peerB send 2, 5 + data of 0
which makes peerA reproduce 3 to verify the content
```

### Scenario B

peerA doesn't have any parent or ancestor tree nodes for the data block.
peerB sends a signature for the widest tree it has that cover the data block and all the missing hashes

```
peerB's digest

      3
  1       5
0   2   4   6

peerA's digest

   (empty)

peerA requests 0 which makes peerB send 2, 5 and the signature for 3
and the data of 0 which makes peerA reproduce 3 and verify that the signature
match
```
