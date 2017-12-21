+++
title = "Let's build a cryptocurrency (part 1): Blockchain"
date = "2017-12-20T22:22:22"
+++

The blockchain is the core data structure in most cryptocurrencies. While first popularised by
its use as a distributed ledger with bitcoin, a blockchain can be used for storing
arbitrary data - not just transaction records. There is an aura of mysticism surrounding the word
"blockchain", but in reality it's a rather simple data structure - a 
linked list of chunks of data called *blocks*. The real magic of the blockchain is not in the
data structure itself, but as a way of a maintaining a *decentralised* and *permissionless*
system of *consensus*.

In this article we will build a simple blockchain in OCaml. The code for which is stored under the 
[`part1_blockchain`](https://github.com/eadanfahey/ORaiml/tree/part1_blockchain) branch.

## Blocks

A block stores a chunk of arbitrary data. Bitcoin stores transactions in blocks, but there is no
reason why a block could not store any type of data - plain text, images, audio etc. A simple
block design consists of four components: *timestamp*, *data*, *previous block hash* and
the *nonce*.

{{% figure src="/images/block.png" caption="The contents of a block" %}}

We will implement a block as a record type, defining the `data` parameter to be a `string` for the
time being.

{{% marginnote %}}[`src/block.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/block.ml#L3){{% /marginnote %}}
{{< highlight ocaml >}}
type t = {
  timestamp: float;
  data: string;
  prevhash: string option;
  nonce: int;
} [@@deriving yojson]
{{< /highlight >}}

The first block in a blockchain, which is called the *genesis* block, does not have a
previous block to point to, so we allow the `prevhash` field to be `None` with the `string option` 
type. Also of note is the strange looking `[@@deriving yojson]` syntax. This annotation is from
the [`ppx_derviving_yojson`](https://github.com/ocaml-ppx/ppx_deriving_yojson) library which
automatically generates the functions `to_yojson` and `of_yojson` for converting a data structure
to and from JSON, respectively. These functions will come in useful later when we talk about saving
the blockchain to file. While the code for these functions is automatically generated,
we do have specify the type signature of the functions in the `.mli` file to use them in other
modules.

{{% marginnote %}}[`src/block.mli`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/block.mli#L14){{% /marginnote %}}
{{< highlight ocaml >}}
val of_yojson : Yojson.Safe.json -> (t, string) Result.result

val to_yojson : t -> Yojson.Safe.json
{{< /highlight >}}

### The Block Hash

To link blocks into a chain, and to ensure integrity of the data stored within a block, we need
a way of creating a creating a unique "fingerprint" to identify a block. This can be achieved
using *cryptographic hash function*. 

{{% figure src="/images/hashing.png" caption="Computing the hash of a block." %}}

A hash function takes some data of arbitrary size and produces a seemingly
random string of fixed length. This string serves as a unique{{% sidenote %}}In theory, it is possible for a "collision" to occur - a hash function producing the same hash for different inputs. In 2017, researchers at Google and the CWI research institute in Amsterdam [generated a collision](https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html) with the SHA-1 hashing algorithm. However, the computation required was immense - 6,500 years of CPU time alongside 110 years of GPU time!{{% /sidenote %}}
fingerprint for the data. Like bitcoin, we will use the SHA-256 hash function. The bash
snippet below shows how changing the input to the hash function slightly produces a completely
different hash.

{{< highlight bash >}}
echo "abc" | sha256sum
#=> 80ed7fc26bc0bbb12ef93ae6dbc4bdaae8262e83e65163626e57c24b8fc316a8

echo "abcd" | sha256sum
#=> fc4b5fd6816f75a7c81fc8eaa9499d6a299bd803397166e8c4cf9280b801d62c
{{< /highlight >}}

Another interesting property of cryptographic hash functions is that they are *one-way* - it is easy to
compute the hash of some data, but given a hash it is computationally impractical to find the
original data. This will turn out to be important in the next article when we discuss 
*proof of work* - a way of protecting the integrity of the blockchain.

The `hash` method defined below computes the hash of a block by first serializing it to json,
and then passing the resulting string through the `sha256` function which utilises the 
[`Cryptokit`](https://github.com/xavierleroy/cryptokit) library.

{{% marginnote %}}[`src/block.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/block.ml#L20){{% /marginnote %}}
{{< highlight ocaml >}}
let sha256 s =
  let hasher = Cryptokit.Hash.sha256 () in
  let encode_hex = Cryptokit.Hexa.encode () in
  let hash = Cryptokit.hash_string hasher s in
  Cryptokit.transform_string encode_hex hash


let hash t = to_yojson t |> Yojson.Safe.to_string |> sha256
{{< /highlight >}}

### Creating a block

With bitcoin, the process of creating a new block is called *mining*. The code below implements the
`mine` function, taking the `data` and `prevhash` as parameters and returning a new block.
The `nonce` will play it's role in the mining process when we implement proof of work in an 
upcoming article. For now we will just set it to `0`.

{{% marginnote %}}[`src/block.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/block.ml#L14){{% /marginnote %}}
{{< highlight ocaml >}}
let mine ~data ~prevhash =
  let timestamp = Time.now () |> to_epoch in
  let nonce = 0 in
  {timestamp; data; prevhash; nonce}
{{< /highlight >}}

## The Blockchain

As mentioned previously, the blockchain is simply a series of blocks connected by pointing to the
previous block in the chain through the `prevhash` field.

{{% marginnote %}}The chain is formed by referring to the hash of the previous block. For example, the hash of the first block is `a9f26` and the second block forms a link by referencing this hash in its `prevhash` field.{{% /marginnote %}}
{{% figure src="/images/blockchain.png" %}}

Under the hood, we will simply implement a blockchain as a list of blocks. As with the
implementation of the block data structure, the type is annotated with `[@@deriving yojson]` to
automatically generate the `to_yojson` and `of_yojson` serialization functions.

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/blockchain.ml#L4){{% /marginnote %}}
{{< highlight ocaml >}}
type t = Block.t list [@@deriving yojson]
{{< /highlight >}}

### Retrieving and adding blocks

When adding a block to the chain we must first check if the block is valid. For the moment, the
new block must satisfy two constraints. Firstly, the `prevhash` of the new block must match 
the hash of the last block in the chain. Furthermore, the `timestamp` of the new block must be 
greater than that of the last block in the chain. These constraints are enforced by the 
`validate_block` function defined below.

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/blockchain.ml#L10){{% /marginnote %}}
{{< highlight ocaml >}}
let validate_block curr_block prev_block =
  let open Block in
  if curr_block.prevhash <> Some (hash prev_block) then
    Error "Invalid previous block hash"
  else if prev_block.timestamp > curr_block.timestamp then
    Error "Invalid timestamp"
  else
    Ok ()
{{< /highlight >}}

Using this function, the `add_block` function is defined as:

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/blockchain.ml#L23){{% /marginnote %}}
{{< highlight ocaml >}}
let add_block t block =
  let open Block in
  match t with
  | [] -> (match is_genesis block with
    | true  -> Ok [block]
    | false -> Error "First block cannot have a previous block hash")
  | hd :: tl -> (match validate_block block hd with
    | Ok () -> Ok (block :: t)
| Error e -> Error e)
{{< /highlight >}}

When the blockchain is empty, we check if the new block is a genesis block with the `is_genesis`
function, otherwise the validity of the new block is checked with `validate_block`.

The `get_block` function, defined below, iterates through the list, finding the first block
whose hash matches the `hash` parameter passed to the function.

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/blockchain.ml#L37){{% /marginnote %}}
{{< highlight ocaml >}}
let get_block t ~hash = List.find t (fun b -> Block.hash b = hash)
{{< /highlight >}}

One last thing before we finish our first implementation of a blockchain - we need a way of
constructing an empty blockchain. Since our blockchain is defined as a list of blocks,
the `empty` function just returns an empty list.

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/blockchain.ml#L37){{% /marginnote %}}
{{< highlight ocaml >}}
let empty = []
{{< /highlight >}}

## Wrapping things up

We have finished implementing the basic functionality of our blockchain. Let's test it out by
creating a new blockchain and adding some blocks to it.

{{% marginnote %}}[`src/main.ml`](https://github.com/eadanfahey/ORaiml/blob/part1_blockchain/src/main.ml)<br><br>The `add_block` function has the type signature `Blockchain.t -> Block.t -> (Blockchain.t, string) result`. Instead of de-structuring the result type to get at the blockchain with a nested pattern match, the `>>=` operator allows us to chain the calls to `add_block` together. If any of the calls to `add_block` fails, the `Err` type returned passes through to the end of the chain. Try adding `block2` first to see how this works. The ["Error Handling"](https://realworldocaml.org/v1/en/html/error-handling.html) chapter of the book Real World OCaml has a good explanation of this concept.{{% /marginnote %}}
{{< highlight ocaml >}}
open Core
open Result

let () =
  let open Block in
  let block1 = mine "Infinite Jest" None in
  let block2 = mine "Gravity's Rainbow" (Some (hash block1)) in
  let blockchain = Blockchain.(
      add_block empty block1 >>= fun bc ->
      add_block bc block2
    ) in
  match blockchain with
  | Error e -> failwith e
  | Ok bc   -> Blockchain.to_yojson bc
               |> Yojson.Safe.pretty_to_string
               |> print_string;

  printf "\n\nHash of block1: %s" (hash block1);
{{< /highlight >}}

The repository contains a `Makefile` for compiling the project using `ocamlbuild`. 
To compile the project to a native executable run `make native`. The main module can then be
run by calling `./main.native`, and you should see output like:

```
[
  {
    "timestamp": 1513810190.1664498,
    "data": "Gravity's Rainbow",
    "prevhash":
      "49ba6f3a15e86699f70bf49d14702a03a4133f5afb3f61b7eb9fb05cf059bff8",
    "nonce": 0
  },
  {
    "timestamp": 1513810190.166321,
    "data": "Infinite Jest",
    "prevhash": null,
    "nonce": 0
  }
]

Hash of block1: 49ba6f3a15e86699f70bf49d14702a03a4133f5afb3f61b7eb9fb05cf059bff8
```

Our implementation of the blockchain so far is fine. However, it is trivial for a malicious actor
to alter the data in the blockchain. Let's run through a scenario to understand this issue
more clearly.

## Attacking the blockchain

{{% figure type="margin" src="/images/characters.png" caption="These are the three characters in our story." %}} 
Suppose there are three people - Alice, Bob and Carol. Alice choses to store the names of her
favourite books in a blockchain. Like bitcoin, this blockchain is distributed globally across a
network of computers and aims to serve as global source of truth.

{{% figure src="/images/alice-blockchain.png" caption="Alice downloads a copy of the blockchain from the network. Her favourite books are Catch-22 and 1984." %}}

Carol also downloads a copy of the blockchain; however, she decides to change one of Alice's books 
from "Catch-22" to "Brave New World". In doing so, she changes the hash of `Block 1` from `3rtx7` to
`rf81a`. But, the previous block hash of `Block 2` still points to the old hash - `3rtx7` - and so,
the blockchain has been split in two.

{{% figure src="/images/carol-invalid-blockchain.png" caption="Carol has split her blockchain in two!"%}}

Carol wants to convince Bob that Brave New World really is one of Alice's favourite books. She has 
two options:

  1. Tell Bob that the lower portion is the true state of the world, pretending that the latter
     portion of the blockchain never existed.
  2. Somehow, join the the two chains back together.
  
The first option is unlikely to pan out in a decentralised network. Like all good blockchain users
Bob is cautious. When downloading the blockchain he doesn't just take Carol's word, but also asks
other nodes in the network about their copy of the blockchain. On discovering that others have a
longer blockchain, there is no reason for Bob to take Carol's blockchain as the truth.
To deceive Bob, Alice must go for option two - join the two chains back together.

And so, Carol begins the process of joining the chains. First she updates the previous block hash 
of `Block 2` to point to the hash of the modified `Block 1`.
This will change the hash of `Block 2` {{% sidenote %}} From `4aqvz` to `6ib2t` in our example.{{% /sidenote %}},
and so will require the previous block hash of `Block 3` to be updated, and so on until the end
of the chain. With the current design of our blockchain, it is trivial for Carol to make these
changes.

{{% figure src="/images/carol-valid-blockchain.png" caption="Carol's blockchain is just as long as that of all other users on the network."%}}

{{% figure type="margin" src="/images/trust.png" caption="Who should bob believe?" %}} 
Now that Carol's blockchain is just as everyone else's copy of the blockchain, who should Bob 
believe? Carol is presenting him with a perfectly valid blockchain which he has no reason to reject.
The fact that it is so easy to overwrite the contents of the blockchain presents a severe flaw in
our current design. A decentralised cryptocurrency where anyone can easily overwrite the 
transaction history is not very useful. In the next article we will modify the design of blockchain,
incorporating a feature called "*proof of work*" which makes altering the contents of a blockchain
practically impossible while maintaining the trust of the network.
