+++
draft = true
title = "Let's build a cryptocurrency (part 2): Proof of Work"
date = "2017-10-30"
math = true
+++

In the [previous article](../crypto1) we implemented a simple blockchain. We discovered a fatal
flaw with our design - it is trivial for someone to overwrite the contents of the blockchain. This
limitation makes our current design entirely unsuitable for a distributed cryptocurrency. In this
article we will overcome this limitation by slightly modifying the design with a feature called 
*proof-of-work*. The key idea is to make it computationally impractical for an attacker to overwrite
the contents of the blockchain.

The code for this article is stored under the
[`part2_pow`](https://github.com/eadanfahey/raicoin/tree/part2_pow) git branch.
The updated codebase can be compared to the the code from part 1 
[here](https://github.com/eadanfahey/raicoin/compare/part1_blockchain...part2_pow).

## Proof-of-work: solving a puzzle

Recall that we call the process of adding a new block to the blockchain as *mining* and the entity
which performs this process is called a *miner*. Proof-of-work algorithms require the miner to
solve a puzzle with the following properties:

  1. The solution to the puzzle should depend on the contents of the block. Furthermore, changing
     the contents of a block should require a new puzzle to be solved.
  2. Solving the puzzle should be computationally difficult.
  3. Proving a solution to the puzzle is valid should be trivial.
  
The proof-of-work algorithm used in bitcoin takes advantage of the properties of cryptographic
hash functions. These properties, which we learned about in the previous article, are ideal for 
creating a proof-of-work algorithm, namely:

  1. Changing the input to a hash function produces a completely different hash.
  2. Given a hash, it is computationally difficult to retrieve the original data.
  3. However, computing the hash of some data is computationally easy in comparison.

Recall, each block has a field called the *nonce*. We ignored this
in the previous article, but now we will discover the role it plays in the proof-of-work algorithm
used by bitcoin.

{{% figure src="/images/pow-algorithm.png" caption="The algorithm for finding a solution to the proof-of-work puzzle - i.e. mining." %}}

Specifically, this proof-of-work algorithm requires finding a value for the nonce such that the
SHA256 hash of the block is less than{{%sidenote%}}Typically a hash is presented as a string, but this is just it's hexadecimal representation. For example, try running this in python: `int(hashlib.sha256(b'hello').hexdigest(), 16)` to see the more familiar base 10 representation.{{%/sidenote%}}
an agreed upon target value. Due to the features of cryptographic hash functions
we learned about, the only way to find a solution is by basically randomly picking a nonce value
and checking if the hash of the block is less than the target value.

The code snippet below shows an updated version of our `mine` method for creating a new block
which finds a solution to the proof-of-work puzzle.

{{% marginnote %}}[`src/block.rs`](https://github.com/eadanfahey/raicoin/blob/part2_pow/src/block.rs#L28){{% /marginnote %}}
{{< highlight rust >}}
impl Block {
    ...

    pub fn mine(data: String, prev_block_hash: String) -> Block {
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;

        let mut block = Block {
            timestamp,
            data,
            prev_block_hash,
            nonce: 0,
        };

        let target = BigInt::one() << (256 - DIFFICULTY);

        loop {
            let hash_int = BigInt::from_str_radix(&block.hash(), 16).unwrap();
            if hash_int < target {
                break;
            } else {
                block.nonce += 1
            }
        }

        block
    }
}
{{< /highlight >}}

The main addition is within the `loop` block where we check if the current value of the nonce
permits a solution to the puzzle and if it doesn't we increment and check again. The size of the
`target` is controlled by the `DIFFICULTY` constant defined in
[`src/constants.rs`](https://github.com/eadanfahey/raicoin/blob/part2_pow/src/constants.rs). The
larger its value the more iterations are required{{%sidenote%}}The bitcoin mining difficulty is updated periodically such that a new block is "found" about every ten minutes.{{%/sidenote%}}
, on average, to find a valid nonce.

When the network is presented with a new block it must check that it's nonce is indeed valid. 
Therefore, we must also update the `validate_block` method of `Blockchain`:

{{% marginnote %}}[`src/blockchain.rs`](https://github.com/eadanfahey/raicoin/blob/part2_pow/src/blockchain.rs#L19){{% /marginnote %}}
{{< highlight rust >}}
impl Blockchain {
    ...
    
    fn validate_block(&self, block: &Block) {
        let target = BigInt::one() << (256 - DIFFICULTY);
        let hash_int = BigInt::from_str_radix(&block.hash(), 16).unwrap();

        if block.prev_block_hash != self.last_block_hash {
            panic!("Error: invalid previous_block_hash")
        } else if hash_int > target {
            panic!("Error: invalid nonce")
        }
    }
    ...
}
{{< /highlight >}}

You can try out the new code by executing `cargo run`{{%sidenote%}}Ensure you are on the `part2_pow` branch.{{%/sidenote%}}
in the terminal. You should see something like:

```
==============================

hash: 000988285ce5a8c334b5d733331630d171dda1ccf8f2bf6636a39f5b33f22fe1
contents: {
  "timestamp": 1511112426,
  "data": "Gravity's Rainbow",
  "prev_block_hash": "000be610a265339326663933274b7f223a4d4e13b01cfc76e5060701c468524d",
  "nonce": 6728
}

==============================

hash: 000be610a265339326663933274b7f223a4d4e13b01cfc76e5060701c468524d
contents: {
  "timestamp": 1511112426,
  "data": "Infinte Jest",
  "prev_block_hash": "0009ff95bd1ba0d07c9e61ea3aad3b6c105fcfd14646529bc974cf623ea09d27",
  "nonce": 678
}

==============================

hash: 0009ff95bd1ba0d07c9e61ea3aad3b6c105fcfd14646529bc974cf623ea09d27
contents: {
  "timestamp": 1511112425,
  "data": "Genesis block",
  "prev_block_hash": "",
  "nonce": 3812
}
```

The code in `main.rs` is the same as in part 1; however, notice that the `nonce` field is no 
longer zero, but a valid solution to the proof-of-work puzzle. Try increasing the 
`DIFFICULTY` constant in `constants.rs` and you should observe that the mining algorithm
takes longer{{%sidenote%}}You may want to turn on compiler optimisations with `cargo run --release`{{%/sidenote%}} to find a valid nonce.

## How does proof-of-work help?

Recall from the previous article that we had three characters - Alice, Bob and Carol. Alice was using
the blockchain to store the names of her favourite books; however, Carol was easily able to
modify the blockchain leaving Bob helpless in deciding whether he should put his faith in the
state of the blockchain according to Alice or that of Carol. However, the new requirement of
finding a solution to the proof-of-work puzzle will make Carol's life much more difficult.

{{% figure src="/images/carol-invalid-blockchain.png" caption="As before, Carol splits her chain in two after modifying the data in block 1 from Catch-22 to Brave New World."%}}

Previously when Carol changed the contents of a block, and thus splitting the chain in two, it was
trivial for her to re-join the chains. But now, when she updates the previous block hash of block
2 from `3rtx7` to `r581a` to begin re-joining the chains she must also find a new value for the
nonce to satisfy the proof-of-work. I have set the mining difficulty quite low so
that it only takes a few seconds to find a valid nonce on my laptop; however, the mining difficulty
is much higher with bitcoin. As of November 2017, it takes about $10^{21}$ iterations, on 
average, to find a valid nonce - about 100,000 years on a laptop! Furthermore, not only does Carol
need to re-mine the block that she modified, she also needs to re-mine all blocks after it in the
chain as they all require the previous block hash parameter to be updated. On top of this,
she must do all of this mining before anyone else mines a new block lest she end up with a
blockchain shorter than the rest of the network{{%sidenote%}}As a rule of thumb, bitcoin users say a transaction is practically impossible to modify once 6 blocks have been placed on top of it. Each new block that is mined and appended to the chain adds further security to the blockchain.{{%/sidenote%}}.

## Conclusion

In this article we have learned how the proof-of-work concept secures the blockchain by requiring
miners to solve a computationally difficult puzzle. Proof-of-work enables a distributed, 
permissionless and transparent method of *consensus*. This is the most amazing innovation spawned
by bitcoin - consensus with any middlemen or trusted third-parties.

Before we discuss how a cryptocurrency can be built on top of a blockchain secured through
proof-of-work, we will build a commandline interface to the blockchain and write some code to
persist the state of the blockchain.
