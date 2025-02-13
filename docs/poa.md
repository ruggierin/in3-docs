## Verifying Blockheaders

Since all proofs always include the blockheader, it is crucial to verify the correctness of these data as well. But verification depends on the consensus of the underlying blockchain. (For details, see [Ethereum Verification and MerkleProof](./Ethereum-Verification-and-MerkleProof).)

```eval_rst
.. graphviz::

  digraph minimal_nonplanar_graphs {
    node [style=filled  fontname="Helvetica"]
  fontname="Helvetica"
  edge[ fontname="Helvetica"]

  subgraph cluster_pow {
    label="Proof or Work"  color=lightblue  style=filled
    node [color=white]

    c[label="Client"]

    A[label="Node A"]
    B[label="Node B"]
    C[label="Node C"]

    c -> B[label=""]
    B -> c[label=" response\n + proof  \n + signed\n    header"]
    B -> A[label=" sign"]
    B -> C

  
  }

  subgraph cluster_poa {
    label="Proof of Authority"  color=lightblue  style=filled
    node [color=white]

    _c[label="Client"]

    _B[label="Node"]

    _c -> _B[label=""]
    _B -> _c[label=" response\n + proof  \n + header"]
  }

  subgraph cluster_pos {
    label="Proof of Stake"  color=lightblue  style=filled
    node [color=white]
  rank=same x N V
      
    x[label="Client"]

    N[label="Node"]
    V[label="Node (Validator)"]

    x -> N[label=""]
    N -> x[label=" response\n + proof  \n + header"]

    x -> V[label=" header"]

  
  }

  }


```

### Proof of Work

Currently, the public chain uses Proof of Work. This makes it very hard to verify the header, since anybody can produce such a header. So, the only way to verify that the block in question is an accepted block is to let registered nodes sign the blockhash. If they are wrong, they lose their previously stored deposit. For the client, this means that the required security depends on the deposit stored by the nodes.
That's why a client may be configured to require multiple signatures and even a minimal deposit:

```js
client.sendRPC('eth_getBalance', [account, 'latest'], chain, {
  minDeposit: web3.utils.toWei(10,'ether'),
  signatureCount: 3
})
```

The `minDeposit` lets the client preselect only nodes with at least that much deposit.
The `signatureCount` asks for multiple signatures and so increases the security.

Since most clients are small devices with limited bandwith, the client is not asking for the signatures directly from the nodes, but chooses one node and lets this node run a subrequest to get the signatures. This not only means fewer requests for the clients, but also that at least one node checks the signatures and "convicts" another if it lied.

### Proof of Authority

The good thing about Proof of Authority is that there is already a signature included in the blockheader. So, if we know who is allowed to sign a block, we do not need an additional blockhash signed. The only critical information we rely on is the list of validators.

Currently, there are two consensus algorithms:

#### Aura

Aura is used by Parity only and there are two ways to configure such:

- **static list of nodes** (like the Kovan-network) - in this case, the validatorlist is included in the chain-spec and cannot change, which makes it very easy for a client to verify blockheaders.
- **validator contract** - a contract that offers a function `getValidators()`. Depending on the chain, this contract may contain rules that define how validators may change. But this flexibility comes with a price. It makes it harder for a client to find a secure way to detect validator changes. That's why the proof for such a contract depends on the rules laid out in the contract and is chain-specific. 

#### Clique

Clique is a protocol developed by the Geth-team and is now also supported by Parity (see Görli-testnet).
Instead of relying on a contract, Clique defines a protocol of how validator nodes may change. All votes are done directly in the blockheader. This makes it easier to prove, since it does not rely on any contract. 

The Incubed nodes will check all the blocks for votes and create a `validatorlist` that defines the validatorset for any given blocknumber. This also includes the proof in form of all blockheaders that either voted the new node in or out. This way, the client can ask for the list and automatically update the internal list after it has verified each blockheader and vote. Even though malicious nodes cannot forge the signatures of a validator, they may skip votes in the validatorlist. That is why a validatorlist update should always be done by running multiple requests and merging them together.

