# Execution State Network

This document is the specification for the sub-protocol that supports on-demand availability of state data from the execution chain.


## Overview

The execution state network is a [Kademlia](https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf) DHT that uses the [Portal Wire Protocol](./portal-wire-protocol.md) to establish an overlay network on top of the [Discovery v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5-wire.md) protocol.

State data from the execution chain consists of all account data from the main storage trie, all contract storage data from all of the individual contract storage tries, and the individul bytecodes for all contracts.

### Data

All of the execution layer state data is stored in two different formats.

- Raw trie nodes
- Leaf data with merkle proof

This duplication is present to enable fast O(1) lookups of leaf information while still allowing one to fall back to the raw trie node data in cases where an exclusion proof must be generated.

#### Types

The network stores the full execution layer state which emcompases the following:

- Account trie and leaf nodes.
- Account leaf nodes with accompanying trie proof.
- Contract storage trie and leaf nodes.
- Contract storage leaf nodes with accompanying trie proof.
- Contract bytecode


#### Retrieval

- Account trie leaf data by account address and state root.
- Contract storage leaf data by account address, state root, and slot number.
- Account trie leaf and intermediate nodes by address, trie path, and node hash.
- Contract storage trie leaf and intermediate nodes by address, trie path, and node hash.
- Contract bytecode by address and code hash.

## Specification

<!-- This section is where the actual technical specification is written -->

### Distance Function

The state network uses the following "ring geometry" distance function.

```python
MODULO = 2**256
MID = 2**255

def distance(node_id: uint256, content_id: uint256) -> uint256:
    """
    A distance function for determining proximity between a node and content.

    Treats the keyspace as if it wraps around on both ends and
    returns the minimum distance needed to traverse between two
    different keys.

    Examples:

    >>> assert distance(10, 10) == 0
    >>> assert distance(5, 2**256 - 1) == 6
    >>> assert distance(2**256 - 1, 6) == 7
    >>> assert distance(5, 1) == 4
    >>> assert distance(1, 5) == 4
    >>> assert distance(0, 2**255) == 2**255
    >>> assert distance(0, 2**255 + 1) == 2**255 - 1
    """
    if node_id > content_id:
        diff = node_id - content_id
    else:
        diff = content_id - node_id

    if diff > MID:
        return MODULO - diff
    else:
        return diff

```

This distance function is designed to preserve locality of leaf data within main account trie and the individual contract storage tries.  The term "locality" in this context means that two trie nodes which are adjacent to each other in the trie will also be adjacent to each other in the DHT.


### Content ID Derivation Function

The derivation function for Content ID values is defined separately for each data type.


### Wire Protocol

#### Protocol Identifier

As specified in the [Protocol identifiers](./portal-wire-protocol.md#protocol-identifiers) section of the Portal wire protocol, the `protocol` field in the `TALKREQ` message **MUST** contain the value of `0x500A`.


#### Supported Message Types

The execution state network supports the following protocol messages:

- `Ping` - `Pong`
- `Find Nodes` - `Nodes`
- `Find Content` - `Found Content`
- `Offer` - `Accept`

#### `Ping.custom_data` & `Pong.custom_data`

In the execution state network the `custom_payload` field of the `Ping` and `Pong` messages is the serialization of an SSZ Container specified as `custom_data`:

```
custom_data = Container(data_radius: uint256)
custom_payload = SSZ.serialize(custom_data)
```


### Routing Table 

The execution state network uses the standard routing table structure from the Portal Wire Protocol.


### Node State

#### Data Radius

The execution state network includes one additional piece of node state that should be tracked.  Nodes must track the `data_radius` from the Ping and Pong messages for other nodes in the network.  This value is a 256 bit integer and represents the data that a node is "interested" in.  We define the following function to determine whether node in the network should be interested in a piece of content.

```
interested(node, content) = distance(node.id, content.id) <= node.radius
```

A node is expected to maintain `radius` information for each node in its local node table. A node's `radius` value may fluctuate as the contents of its local key-value store change.

A node should track their own radius value and provide this value in all Ping or Pong messages it sends to other nodes.


### Data Types

#### Nibbles

We define a custom SSZ sedes alias `Nibbles` to mean `List[uint8, max_length=64]` where each individual value **must** be constrained to a valid "nibbles" value of `0 - 15`.


#### Account Trie Node (0x00)

An individual trie node from the main account trie.

```
account_trie_node_key := Container(path: Nibbles, node_hash: Bytes32, state_root: Bytes32)
content               := RLP.encode(trie_node)
selector              := 0x00

content_id            := sha256(path | node_hash)
content_key           := selector + SSZ.serialize(account_trie_node_key)
```

#### Contract Storage Trie Node

An individual trie node from a contract storage trie.

```
storage_trie_node_key := Container(address: Bytes20, path: Nibbles, node_hash: Bytes32, state_root: Bytes32)
selector              := 0x01

content_id            := sha256(address | path | node_hash)
content_key           := selector + SSZ.serialize(storage_trie_node_key)
```

#### Account Trie Proof

A leaf node from the main account trie and accompanying merkle proof against a recent `Header.state_root`

```
account_trie_proof_key := Container(address: Bytes20, state_root: Bytes32)
selector               := 0x02

content_id             := keccak(address)
content_key            := selector + SSZ.serialize(account_trie_proof_key)
```

#### Contract Storage Trie Proof

A leaf node from a contract storage trie and accompanying merkle proof against the `Account.storage_root`.

```
storage_trie_proof_key := Container(address: Bytes20, slot: uint256, state_root: Bytes32)
selector               := 0x03

content_id             := (keccak(address) + keccak(slot)) % 2**256
content_key            := selector + SSZ.serialize(storage_trie_proof_key)
```

#### Contract Bytecode

The bytecode for a specific contract as referenced by `Account.code_hash`

```
contract_bytecode_key := Container(address: Bytes20, code_hash: Bytes32)
selector              := 0x04

content_id            := sha256(address | code_hash)
content_key           := selector + SSZ.serialize(contract_bytecode_key)
```


## Gossip

The state network will use a multi stage mechanism to store new and updated state data.

### Overview

The state data is stored in the network in two formats.

- A: Trie nodes
    - Individual leaf and intermediate trie nodes.
    - Gossiped with a proof against a recent state root
    - Optionally stored without a proof not anchored to a specific state root
- B: Leaf proofs
    - Contiguous sections of leaf data
    - Gossiped and stored with a proof against a specific state root
    - Proof is continually updated to latest state root

The state data first enters the network as trie nodes which are then used to populate and update the leaf proofs.

### Terminology

We define the following terms when referring to state data.

> The diagrams below use a binary trie for visual simplicity. The same
> definitions naturally extend to the hexary patricia trie.


```
0:                           X
                            / \
                          /     \
                        /         \
                      /             \
                    /                 \
                  /                     \
                /                         \
1:             0                           1
             /   \                       /   \
           /       \                   /       \
         /           \               /           \
2:      0             1             0             1
       / \           / \           / \           / \
      /   \         /   \         /   \         /   \
3:   0     1       0     1       0     1       0     1
    / \   / \     / \   / \     / \   / \     / \   / \
4: 0   1 0   1   0   1 0   1   0   1 0   1   0   1 0   1
```


#### *"state root"*

The node labeled `X` in the diagram.

#### *"trie node"*

Any of the individual nodes in the trie.

#### *"intermediate node"*

Any of the nodes in the trie which are computed from other nodes in the trie.  The nodes in the diagram at levels 0, 1, 2, and 3 are all intermediate.

#### *"leaf node"*

Any node in the trie that represents a value stored in the trie.  The nodes in the diagram at level 4 are leaf nodes.

#### *"leaf proof"*

The merkle proof which contains a leaf node and the intermediate trie nodes necessary to compute the state root of the trie.

### Stages

The gossip mechanism is divided up into individual stages which are designed to
ensure that each individual piece of data is properly disseminated to the DHT
nodes responsible for storing it, as well as spreading the responsibility as
evenly as possible across the DHT.

The stages are:

- Stage 1:
    - Bridge node generates a proof of all new and updated state data from the most recent block and initiates gossip of the individual trie nodes.
- Stage 2:
    - DHT nodes receiving trie nodes perform [neighborhood gossip](./portal-wire-protocol.md#neighborhood-gossip) to spread the data to nearby interested DHT nodes.
    - DHT nodes receiving trie nodes extract the trie nodes from the anchor proof to perform *recursive gossip* (defined below).
    - DHT nodes receiving "leaf" nodes initiate gossip of the leaf proofs (for stage 3)
- Stage 3:
    - DHT nodes receiving leaf proofs perform *neighborhood gossip* to spread the data to nearby interested DHT nodes.


```
    +-------------------------+
    | Stage 1: data ingress   |
    +-------------------------+
    |                         |
    | Bridge node initializes |
    | trie node gossip        |
    |                         |
    +-------------------------+
            |
            v
    +---------------------------+
    | Stage 2: trie nodes       |
    +---------------------------+
    |                           |
    | A: neighborhood gossip of |
    |    trie node and proof    |
    |                           |
    | B: initialization of      |
    |    gossip for proof trie  |
    |    nodes.                 |
    |                           |
    | C: initialization of      |
    |    leaf proof gossip      |
    |                           |
    +---------------------------+
            |
            v
    +----------------------------+
    | Stage 3: leaf proofs       |
    +----------------------------+
    |                            |
    | neighborhood gossip of     |
    | leaf proofs                |
    |                            |
    +----------------------------+

```

The phrase "initialization of XXX gossip" refers to finding the DHT nodes that
are responsible for XXX and offering the data to them.


#### Stage 1: Data Ingress


The first stage of gossip is performed by a bridge node. Each time a new block
is added to their view of the chain, a set of merkle proofs which are all
anchored to `Header.state_root` is generated which contains.

- Account trie Data:
    - All of the intermediate and leaf trie nodes from the account trie necessary to prove new and modified accounts.
    - All of the intermediate and leaf trie nodes from the account trie necessary for exclusion proofs for deleted accounts.
- Contract Storage trie data:
    - All of the intermediate and leaf trie nodes from each contract storage trie necessary to prove new and modified storage slots.
    - All of the intermediate and leaf trie nodes from each contract storage trie necessary for exclusion proofs for zero'd storage slots.
- All contract bytecode for newly created contracts

> TODO: Figure out language for defining which trie nodes from this proof the bridge node must initialize gossip.

> TODO: Determine mechanism for contract code.


#### Stage 2A: Neighborhood Trie Node Gossip


When individual trie nodes are gossiped they will be transmitted as both the trie node itself, and the additional proof nodes necessary to prove the trie node against a recent state root.

The receiving DHT node will perform *neighborhood* gossip to nearby nodes from their routing table.


#### Stage 2B: Recursive Trie Node Gossip

When individual trie nodes are gossiped, the receiving node is responsible for initializing gossip for other trie nodes contained in the accompanying proof.

This diagram illustrates the proof for the trie node under the path `0011`.

```
0:                           X
                            / \
                          /     \
                        /         \
                      /             \
                    /                 \
                  /                     \
                /                         \
1:             0                           1
             /   \
           /       \
         /           \
2:      0             1
       / \
      /   \
3:   0     1
          / \
4:       0  (1)

```

Note that it also includes the intermediate trie nodes under the paths:

- `000`
- `001`
- `00`
- `01`
- `0`
- `1`
- `X`

The gossip payload for the trie node at `0011` will contain these trie nodes as a proof.  Upon receipt of this trie node and proof, the receiving DHT node will extract the proof for the intermediate node under the path `001` which is the direct parent of the main trie node currently being gossiped.  This proof would be visualized as follows:


```
0:                           X
                            / \
                          /     \
                        /         \
                      /             \
                    /                 \
                  /                     \
                /                         \
1:             0                           1
             /   \
           /       \
         /           \
2:      0             1
       / \
      /   \
3:   0    (1)

4:

```

The DHT node would then *initialize* gossip for the trie node under the path `001`, using this new slimmed down proof to anchor to the same state root.  This process continues until it reaches the state root where it naturally terminates.  We refer to this as *recursive trie node* gossip.

#### Stage 2C: Initialization Of Leaf Proof Gossip

When receiving an individual trie node which represents a "leaf" node in the trie, the combined leaf node and proof are equivalent to a *"leaf proof"*.  The DHT node receiving this data in the form of "trie node" data is responsible for *initializing gossip* for the same data as a *"leaf proof"*.

#### Stage 3: Neighborhood *"Leaf Proof"* Gossip

When receiving a *"leaf proof"* over gossip, a DHT node will perform *"neighborhood gossip"* to nearby nodes from their routing table.


### Updating cold Leaf Proofs

Anytime the state root changes for either the main account trie or a contract storage trie, every leaf proof under that root will need to be updated.  The primary gossip mechanism will ensure that leaf data that was added, modified, or removed will receive and updated proof.  However, we need a mechanism for updating the leaf proofs for "cold" data that has not been changed.

Each time a new block is added to the chain, the DHT nodes storing leaf proof data will need to perform a walk of the trie starting at the state root. This walk of the trie will be directed towards the slice of the trie dictated by the set of leaves that the node is storing. As the trie is walked it should be compared to the previous proof from the previous state root. This walk concludes once all of the in-range leaves can be proven with the new state root.


> TODO: reverse diffs and storing only the latest proof.

> TODO: gossiping proof updates to neighbors to reduce duplicate work.
