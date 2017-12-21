+++
title = "Let's build a cryptocurrency (part 2): Proof of Work"
date = "2017-12-21"
math = true
+++

In the [previous article](../crypto1) we implemented a simple blockchain. We discovered a fatal
flaw with our design - it is trivial for someone to overwrite the contents of the blockchain. This
limitation makes our current design entirely unsuitable for a distributed cryptocurrency. In this
article we will overcome this limitation by slightly modifying the design with a feature called 
*proof-of-work*. The key idea is to make it computationally impractical for an attacker to overwrite
the contents of the blockchain.

The code for this article is stored under the
[`part2_pow`](https://github.com/eadanfahey/ORaiml/tree/part2_pow) git branch.
The updated codebase can be compared to the the code from part 1 
[here](https://github.com/eadanfahey/ORaiml/compare/part1_blockchain...part2_pow).

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
an agreed upon target value. Due to the features of cryptographic hash functions that
we learned about in previously, the only way to find a nonce to solve the puzzle is by brute force 
iteration through the set of positive integers.

The code snippet below shows an updated version of the `mine` function from the previous article
for solving the proof-of-work puzzle.

{{% marginnote %}}[`src/block.ml`](https://github.com/eadanfahey/ORaiml/blob/part2_pow/src/block.ml#L25)<br><br>Since SHA256 outputs a 256 bit integer, we use [`bignum`](https://github.com/janestreet/bignum) library to get the integer representation of the block hash from its hexadecimal form.{{% /marginnote %}}
{{< highlight ocaml >}}
let mine ~data ~prevhash =
  let timestamp = Time.now () |> to_epoch in
  let rec mine_loop block =
    let hash_int = Bigint.Hex.of_string ("0x" ^ (hash block)) in
    match Bigint.(hash_int < Constants.pow_target) with
    | true  -> block
    | false -> mine_loop {block with nonce = block.nonce + 1}
  in
  mine_loop {timestamp; data; prevhash; nonce = 0}
{{< /highlight >}}

The main addition is the `mine_loop` function that finds a nonce to solve the proof-of-work 
puzzle. Starting with the nonce set to zero, `mine_loop` recursively calls itself until the hash
of the block is less than the target value. The target value - `pow_target` - is defined in
`src/constants.ml`:

{{% marginnote %}}[`src/constants.ml`](https://github.com/eadanfahey/ORaiml/blob/part2_pow/src/constants.ml){{% /marginnote %}}
{{< highlight ocaml >}}
open Bignum

let pow_difficulty = 12

let pow_target =
  let i = Bigint.of_int (256 - pow_difficulty) in
  let two = Bigint.of_int 2 in
  Bigint.pow two i
{{< /highlight >}}

The computational difficulty of mining is controlled by the `pow_difficulty` constant. The target
value is calculated as $2^{256 - \text{pow_difficulty}}$, and so the larger the difficulty, the
more iterations are required, on average, to find a valid nonce.

When the network is presented with a new block it must check that its nonce is indeed valid. 
Therefore, we must also update the `validate_block` function:

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part2_pow/src/blockchain.ml#L11){{% /marginnote %}}
{{< highlight ocaml >}}
let validate_block curr_block prev_block =
  let open Block in
  let hash_int = Bigint.Hex.of_string ("0x" ^ (hash curr_block)) in
  if curr_block.prevhash <> Some (hash prev_block) then
    Error "Invalid previous block hash"
  else if prev_block.timestamp > curr_block.timestamp then
    Error "Invalid timestamp"
  else if Bigint.(hash_int > Constants.pow_target) then
    Error "Invalid nonce"
  else
    Ok ()
{{< /highlight >}}

As before, you can compile the project to a native executable with 
`make native`{{%sidenote%}}Ensure you are on the `part2_pow` branch.{{%/sidenote%}} and call
`./main.native` to run the project. You should see something like:

```
[
  {
    "timestamp": 1513893342.9278667,
    "data": "Gravity's Rainbow",
    "prevhash":
      "000bb215413cc88dd9fbbe763771175176804261167898a9cdf528e3248fba8f",
    "nonce": 3912
  },
  {
    "timestamp": 1513893342.8890088,
    "data": "Infinite Jest",
    "prevhash": null,
    "nonce": 1424
  }
]

Hash of block1: 000bb215413cc88dd9fbbe763771175176804261167898a9cdf528e3248fba8f
```

The code in `main.ml` is the same as in part 1; however, notice that the `nonce` field is no 
longer zero, but a valid solution to the proof-of-work puzzle. Try increasing the 
`pow_difficulty` constant in `constants.ml` and you should observe that the mining algorithm
takes longer to solve the proof-of-work puzzle.

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
proof-of-work, in the next article we will build a commandline interface to the blockchain and 
write some code to persist the state of the blockchain to file.
