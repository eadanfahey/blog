+++
draft = false
title = "Let's build a cryptocurrency (part 4): Transactions"
date = "2018-01-08"
+++

In the [previous article](../crypto3) we implemented a CLI for our blockchain and added the ability
to persist the state of the chain to file. In this article we will begin building a cryptocurrency
atop the blockchain, starting with an implementation of transactions based on a simplified version
of those found in bitcoin.

The code for this article is stored under the
[`part4_transactions`](https://github.com/eadanfahey/ORaiml/tree/part4_transactions) git branch.
The updated codebase can be compared to the the code from part 3
[here](https://github.com/eadanfahey/ORaiml/compare/part3_cli...part4_transactions).

## Transacting in the traditional banking system

The system for transacting in a cryptocurrency such as bitcoin is very different from how one may
imagine transactions occur in the traditional banking system. To illustrate this, let's bring back
a few of the characters from previous articles. Suppose Alice instructs her bank to
send $20 dollars to Bob.

{{% figure src="/images/bank-transaction.png" caption="A simplified version of a transaction between bank accounts." %}}

We could imagine that the bank first checks that Alice has at least $20 dollars in her bank account,
and if she does, subtract $20 from her account and add it to Bob's account. While this may be a
highly simplified view of how a banking transaction occurs, it illustrates a key requirement of the
traditional banking system - trust in a centralised third party. Depending on your viewpoint, this
has several downsides - Alice must trust that the bank actually has her money{{%sidenote%}}With fractional reserve banking they don't! We saw the effects of this when Cypriot banks had to shut down to prevent a bank run during the financial crisis. https://www.theguardian.com/world/2013/mar/26/cyprus-banks-closed-prevent-run-deposits{{%/sidenote%}},
she must trust the bank to give her access to her money and she must hope that the bank will actually
allow her to send money to Bob.

If only we had a decentralised, permissionless and incredibly secure system for storing financial
transactions - we do, and it's a blockchain secured with proof-of-work!

## Transacting on a blockchain

Unlike transactions in the traditional banking system which we considered above, transactions in
bitcoin work quite differently. For starters, there is no centralised authority. The only rules are
those enforced by the computer code running on the network. Furthermore, there are no accounts 
from which coins can be subtracted from or added to. Instead, the blockchain serves as a *ledger*, 
immutably recording *transactions* between bitcoin *addresses*.

## Transactions

{{% figure type="margin" src="/images/input-output.png" caption="A transaction input and a transaction output." %}}
{{% figure type="margin" src="/images/transaction.png" caption="A transaction is a collection of inputs and outputs. Each input must reference an output from another transaction." %}}

Bitcoin transactions consist of a collection *inputs* and *outputs* satisfying the following rules:

  1. Each input to a transaction must reference an output from another transaction.
  2. An output can be referenced by at most one input.
  3. There can be outputs which are not referenced by an input. These are called *unspent* outputs.
  
The picture below shows an example of how transactions could be linked between inputs and outputs.

{{% figure src="/images/transaction-links.png"%}}

We will begin our implementation by specifying the interface to a standard transaction with the
`StandardTX` module as shown below.

{{% marginnote %}}[`src/transaction.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.mli#L21){{% /marginnote %}}
{{< highlight ocaml >}}
module StandardTX : sig
  type t
  val create: inputs:TXInput.t list -> outputs:TXOutput.t list -> t
  val inputs: t -> TXInput.t list
  val outputs: t -> TXOutput.t list
  val of_yojson: Yojson.Safe.json -> (t, string) result
  val to_yojson: t -> Yojson.Safe.json
end
{{< /highlight >}}

The `StandardTX` type, `t`, is specified as an abstract type constructed by the `create` function
which takes a list of transaction inputs and a list of transaction outputs. We will define the
`TXInput` and `TXOutput` transactions in the next section. The `StandardTX` module also includes
functions `inputs` and `outputs` for accessing the transaction inputs and transaction outputs
stored in the transaction. As usual, we will use the `ppx_deriving` annotation to generate the 
`of_yojson` and `to_yojson` serialisation functions. With the interface to the module specified,
let's look at it's implementation.

{{% marginnote %}}[`src/transaction.ml`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.ml#L23){{% /marginnote %}}
{{< highlight ocaml >}}
module StandardTX = struct
  type t = {
    inputs: TXInput.t list;
    outputs: TXOutput.t list;
  } [@@deriving yojson]
  let create ~inputs ~outputs = {inputs; outputs}
  let inputs t = t.inputs
  let outputs t = t.outputs
end
{{< /highlight >}}

The implementation here is quite straightforward. The abstract type `t` is implemented using a 
record data type storing the transaction input and transaction output lists.

## The contents of inputs and outputs

The sender and recipient in a transaction are specified by the transaction inputs and outputs,
respectively. As mentioned previously, each input references an output. This reference is stored
within the transaction input. The transaction output also stores the value of coins to be given to 
the recipient.
{{% figure src="/images/input-output-data.png"%}}

Similar to the standard transaction implementation, a transaction output is specified as an
abstract type under the `TXOutput` module.

{{% marginnote %}}[`src/transaction.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.mli#L3){{% /marginnote %}}
{{< highlight ocaml >}}
module TXOutput : sig
  type t
  val create: value:float -> recipient:string -> t
  val value: t -> float
  val recipient: t -> string
  val of_yojson: Yojson.Safe.json -> (t, string) result
  val to_yojson: t -> Yojson.Safe.json
end
{{< /highlight >}}

The `create` function of `TXOutput` takes the `value` of coins to send as a `float` and the
`recipient` of those coins as a `string`. The `value` and `recipient` functions are used for
retieving the data stored within the abstract type `t`. The interface to the `TXInput` module,
shown below, is similar.

{{% marginnote %}}[`src/transaction.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.mli#L12){{% /marginnote %}}
{{< highlight ocaml >}}
module TXInput : sig
  type t
  val create: output:TXOutput.t -> sender:string -> t
  val output: t -> TXOutput.t
  val sender: t -> string
  val of_yojson: Yojson.Safe.json -> (t, string) result
  val to_yojson: t -> Yojson.Safe.json
end
{{< /highlight >}}

Instead of a reference to a transaction output, the `create` function of `TXInput` stores a copy
of the output which it is referencing. In contrast, bitcoin references the transaction output by
storing an identifier to locate the transaction in which the output is stored and an index specifying 
where the output is located within that transaction's list of outputs. Although our implementation
is less memory efficient, it is more robust against errors and greatly simplifies the algorithms
for creating and verifying transactions.{{% sidenote %}}In a talk titled ["Making Impossible States Impossible"](https://www.youtube.com/watch?v=IcgmSRJHu_8), Richard Feldman shows how the advanced type system in languages such as OCaml (or in his case [Elm](http://elm-lang.org/)) allow the programmer to encode the desired behaviour into the type system such that errors in the program's logic are impossible. We will try to apply this concept throughout the series.{{% /sidenote %}}
The underlying implementation of the `TXOutput` and `TXInput` interfaces shown above can be seen in 
[src/transaction.ml](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.ml#L3).

## How do transactions work?

We have talked about the data stored in transactions, but we have yet to learn how a transaction
works. To understand this, let's bring back Alice and Bob and let's suppose Alice wants to send
20 coins to Bob. How is this transaction constructed?

{{% figure src="/images/how-transactions-work.png"%}}

First, Alice searches through the set of unspent transaction outputs, also known
as the *UTXO set*, for outputs belonging to her. The total value of all unspent outputs belonging
to Alice is her "balance". To send 20 coins to Bob, Alice must find unspent output(s) with a total
value of at least 20. These unspent outputs are referenced by inputs in a new transaction. The 
transaction will contain an output sending 20 coins to Bob. If the total value of the inputs exceeds
20, the remaining change is given back to Alice in a second output.

## Where do "coins" come from?

We talked about Alice searching the UTXO set for outputs containing belonging to her. The coin 
`value` is stored in these outputs, but where do coins come from?

Recall, that the person which creates a new block by solving the proof-of-work puzzle is called a
miner. Mining a block is computationally expensive, requiring the miner to purchase computers and
the electricity to run them. In exchange for this, the miner is allowed to reward itself with some
coins in a special *coinbase* transaction. Coinbase transactions are different to standard
transactions in that they contain no inputs, just an output containing the mining reward.

Now that we know how transactions work, and the information stored within them, we need a place to
store them. This is where the blockchain comes into play. Previously, we said that each block had
a `data` parameter for storing arbitrary data, and this is where transactions are stored.

To understand how a transaction makes it's way into a block, let's bring back our old friends
Alice and Bob, and introduce a new character - Mario the miner. When Alice wants to send some
bitcoin to Bob, she creates a transaction and propagates it to the network for it to be placed
into a block. Miners and other nodes on the network that come across Alice's transaction put the
transaction in a waiting area called the *mempool*. When Mario decides to start mining a new block
he picks some transactions from his mempool and begins mining a new block. After he solves the
proof-of-work puzzle he sends the block out to the rest of the network. If the network regards his
block as valid, the transactions - along with Mario's mining reward - in the block make their way 
into the blockchain of each network participant.

{{% figure src="/images/mario-mining.png" type="full" %}}

Coming back to our implementation, we have a data structure for standard
transactions - `StandardTX` - so let's make one for coinbase transactions too.

{{% marginnote %}}[`src/transaction.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.mli#L30){{% /marginnote %}}
{{< highlight ocaml >}}
module CoinbaseTX : sig
  type t
  val create: outputs:TXOutput.t list -> t
  val outputs: t -> TXOutput.t list
  val of_yojson: Yojson.Safe.json -> (t, string) result
  val to_yojson: t -> Yojson.Safe.json
end
{{< /highlight >}}

The interface to the `CoinbaseTX` module is similar to that of `StandardTX` we saw earlier. The
only difference being that the `CoinbaseTX` type does not store any transaction inputs. The
underlying implementation of `CoinbaseTX` is in [src/transaction.ml](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.ml#L33).

Finishing off the implementation of the transaction data structures, the main type of the
`Transaction` module specified in `transaction.mli` is defined as either a `StandardTX.t` or a
`CoinbaseTX.t` as shown below.

{{% marginnote %}}[`src/transaction.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.mli#L38){{% /marginnote %}}
{{< highlight ocaml >}}
type t =
  | Standard of StandardTX.t
  | Coinbase of CoinbaseTX.t
{{< /highlight >}}

Instead of storing arbitrary string data, blocks are now used to store transactions. The new version
of the `Block` datatype replaces the `data` field with a `transaction` field storing a list of
transactions. The `mine` function must also be refactored to take a list of transactions rather than
the string data.

{{% marginnote %}}[`src/block.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/block.mli#L3){{% /marginnote %}}
{{< highlight ocaml >}}
type t = {
  timestamp: float;
  transactions: Transaction.t list;
  prevhash: string option;
  nonce: int;
}

(* Mine a block given some data and the previous block hash *)
val mine: transactions:Transaction.t list -> prevhash:string option -> t
{{< /highlight >}}

We have finished specifying and implementing the data structures for creating transactions.
However, we have not specified how to create *valid* transactions to send coins from one user to
another. In the next section we will take the first step towards this goal by implementing the
UTXO set data structure.

## The UTXO set

Recall that the UTXO set is the set of all unspent transaction outputs in the blockchain. There
are two ways we can go about implementing this set:

  1. Scanning through the entire blockchain collecting all unspent transaction outputs each time
     we need the UTXO set.
  2. Maintain the UTXO set alongside the blockchain which we update each time a new block is added
     to the blockchain.
     
We will go for the second option as it saves us from having to compute the UTXO set each time
we need to for example send coins or check someone's balance. I also found it easier to implement
elegantly. Continuing or type driven approach to development, let's specify the interface to the
`Utxo` module.

{{% marginnote %}}[`src/utxo.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/utxo.mli){{% /marginnote %}}
{{< highlight ocaml >}}
type t

val empty: t

val update: t -> Transaction.t -> t

val to_list: t -> Transaction.TXOutput.t list

val of_yojson : Yojson.Safe.json -> (t, string) Result.result

val to_yojson : t -> Yojson.Safe.json
{{< /highlight >}}

The UTXO set is represented by the abstract type `Utxo.t`. The `empty` function returns as you
would expect, an empty UTXO set while the `to_list` function returns a list of the transaction
outputs in the set. The most interesting function here is `update`, which takes a UTXO set and 
a transaction and returns a new UTXO set. The full implementation of the `Utxo` module is shown
below.

{{% marginnote %}}[`src/utxo.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/utxo.ml){{% /marginnote %}}
{{< highlight ocaml >}}
open Core

type t = Transaction.TXOutput.t list [@@deriving yojson]

let empty = []

let to_list t = t

(* Remove the first occurrence of x from the list l, if it exists.*)
let remove l a =
  let rec helper acc rest =
    match rest with
    | [] -> List.rev acc
    | hd :: tl -> if hd = a then List.rev_append acc tl else helper (hd :: acc) tl
  in
  helper [] l

let update t tx =
  let open Transaction in
  let t = List.append t (outputs tx) in
  match tx with
  | Coinbase _ -> t
  | Standard tx -> (
      let ref_outputs = List.map (StandardTX.inputs tx) ~f:(TXInput.output) in
      List.fold ref_outputs ~init:t ~f:remove
)
{{< /highlight >}}

Under the hood, the `Utxo.t` data type is implemented as a list of transaction outputs. As per
usual, the `@@deriving yojson` annotation is used to generate the `of_yojson` and `to_yojson`
serialisation functions. Since the type `t` is just a list, the implementation of `empty` and 
`to_list` are very straightforward. Updating the UTXO set with a new transaction requires us to 
add all outputs of the transaction to the set and then remove from the set all outputs referenced
in the inputs to the transaction. The first step is achieved with the line: 

```ocaml
let t = List.append t (outputs tx)
```

This extracts the transaction outputs from the transaction `tx` with the [`outputs`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/transaction.mli#L46) 
function them to the UTXO list. If the transaction `tx` corresponds to a `Coinbase` transaction, 
we are done since coinbase transactions do not contain any inputs. On the other hand, if `tx`
corresponds to a `Standard` transaction we first extract the outputs referenced in the inputs with:

```ocaml
let ref_outputs = List.map (StandardTX.inputs tx) ~f:(TXInput.output) in
```

and then remove them from the UTXO list `t` with the line:

```ocaml
List.fold ref_outputs ~init:t ~f:remove
```

Here I am making use of the `remove` function that I defined which removes the first occurrence
of a specified element from a list, if it exists. It may be tempting here to filter the UTXO list,
perhaps using the `List.filter` function, removing any output that matches one of those found in 
`ref_outputs`. However, the UTXO set may contain multiple transaction outputs which are the same 
(i.e. they have the same recipient and value). Therefore, we use the `remove` function to only 
remove the *first* occurrence of a matching transaction output from the list.

With the `Utxo` module implemented, let's refactor our blockchain implementation to maintain a valid
UTXO set each time a new block is added to the chain. The interface to the `Blockchain` module
remains the same, except for the addition of the `utxo` function which simply returns the UTXO
set of a blockchain.

{{% marginnote %}}[`src/blockchain.mli`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/blockchain.mli#L13){{% /marginnote %}}
{{< highlight ocaml >}}
(* The UTXO set of the blockchain *)
val utxo: t -> Utxo.t
{{< /highlight >}}

Alongside the list of blocks, the underlying implementation of the `Blockchain.t` abstract type
now also includes it's corresponding UTXO set. Also, when creating a new blockchain with the
`create` function, we now also create an empty UTXO set.

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/blockchain.ml#L5){{% /marginnote %}}
{{< highlight ocaml >}}
type t = {
  blocks: Block.t list;
  utxo: Utxo.t;
} [@@deriving yojson]

let empty = {blocks = []; utxo = Utxo.empty}
{{< /highlight >}}

The main change to the implementation of the `Blockchain` module belongs to its `add_block` 
function.

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/blockchain.ml#L26){{% /marginnote %}}
{{< highlight ocaml >}}
let add_block t block =
  let utxo = List.fold (Block.transactions block) ~init:t.utxo ~f:Utxo.update in
  match t.blocks with
  | [] -> (match is_genesis block with
    | true  -> Ok {blocks = [block]; utxo = utxo}
    | false -> Error "First block cannot have a previous block hash")
  | hd :: tl -> (match validate_block block hd with
    | Ok () -> Ok {blocks = block :: t.blocks; utxo = utxo}
    | Error e -> Error e)
{{< /highlight >}}

A block contains a list of transactions and the UTXO set must be updated to reflect the content
of these transactions when we add a new block to the chain. This update is performed in the
first line of the function:

```ocaml
let utxo = List.fold (Block.transactions block) ~init:t.utxo ~f:Utxo.update in
```

We extract a list of transactions from the block with the `Block.transactions` function and for
each of these transactions, we apply the `Utxo.update` function to update the UTXO set of the
blockchain. The rest of this function is similar to before, only now we return the updated
UTXO, `utxo`, set alongside the blockchain.

Implementing the `Utxo` module has been one of the more complex pieces of code we have seen so far.
But, with the implementation complete, we can use the UTXO set for calculating user balances and
creating transactions for sending coins between users.

## Calculating user balances

Recall, the balance of each user is simply the total value of their unspent transaction outputs.
With the `Utxo` module, the implementation of the `balances` function is quite simple.

{{% marginnote %}}[`src/client.ml`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/client.ml#L3){{% /marginnote %}}
{{< highlight ocaml >}}
let balances bc =
  let open Transaction in
  let utxo = Utxo.to_list (Blockchain.utxo bc) in
  List.fold utxo ~init:String.Map.empty ~f:(fun acc output ->
      let recipient = TXOutput.recipient output in
      let value = TXOutput.value output in
      let newbal = match Map.find acc recipient with
        | None -> value
        | Some bal -> value +. bal in
      Map.set acc recipient newbal
    ) 
{{< /highlight >}}

Here we use the `Utxo.to_list` function to convert the UTXO type to a list of outputs. The balance
of each user is stored in a `Map`, with a mapping from the user's name to their coin balance.

## Sending coins between users

In the section [how do transactions work?](#how-do-transactions-work) we worked through an example
of how a transaction is constructed when Alice sends coins to Bob. The first step in this process
is to search the UTXO set for outputs belonging to the user sending the coins and whose total value
exceeds the desired amount of coins to send. To help us achieve this, I have defined the
`sufficient_outputs` helper function which takes a list of transaction outputs, `txoutputs`, and
the `amount` of coins being sent as input.
If there exists a subset of the outputs list whose total value exceeds that of the `amount` being
sent, this function returns their total value alongside the subset in the tuple `(acc, keep)`. 
Otherwise, if the user's balance is less than the amount being sent, the function returns `None`.

{{% marginnote %}}[`src/client.ml`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/client.ml#L15){{% /marginnote %}}
{{< highlight ocaml >}}
(* Filter a list of outputs such that their total value is sufficient for
   creating a transaction of given amount *)
let sufficient_outputs txoutputs amount =
  let open Transaction in
  let rec helper acc keep rest =
    match acc >= amount with
    | true -> Some (acc, keep)
    | false -> (match rest with
        | [] -> None
        | hd :: tl -> helper (acc +. TXOutput.value hd) (hd :: keep) tl
      )
  in
helper 0.0 [] txoutputs
{{< /highlight >}}

This leads us to the main `send` function which creates a valid transaction to send coins
between users.

{{% marginnote %}}[`src/client.ml`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/client.ml#L29){{% /marginnote %}}
{{< highlight ocaml >}}
let send bc ~sender ~recipient ~amount =
  let open Transaction in
  let sender_outputs =
    Blockchain.utxo bc
    |> Utxo.to_list
    |> List.filter ~f:(fun output -> TXOutput.recipient output = sender) in
  match sufficient_outputs sender_outputs amount with
  | None ->
    Error (sprintf "%s has insufficient funds" sender)
  | Some (total_amount, ref_outputs) -> (
      let new_inputs = List.map ref_outputs ~f:(
          fun out -> TXInput.create out sender) in
      let new_outputs = [
        TXOutput.create amount recipient;
        TXOutput.create (total_amount -. amount) sender
      ] in
      Ok (Standard (StandardTX.create new_inputs new_outputs))
    )
{{< /highlight >}}

We start by filtering the UTXO set for outputs belonging to the `sender` and store the result in
`sender_outputs`. If the sender's balance is less than the desired `amount` to send, we return
an error stating that the user has insufficient funds. Otherwise, we use the outputs returned by
`sufficient_outputs` to create the inputs for a new transaction. The new transaction contains
two outputs - one corresponding the desired `amount` being sent and the other containing
the change sent back to the `sender`.

## Updated Commandline Interface 

Now that we have made the first steps in building a transaction ledger on our blockchain we
should update the blockchain's commandline interface to make use of the `balances` and `send` 
functions we defined above. The updated code for the commandline interface is fairly
straightforward and can be found in
[`src/cli.ml`](https://github.com/eadanfahey/ORaiml/blob/part4_transactions/src/cli.ml).

As usual, you can compile the project to a native exectuable{{%sidenote%}}Remember to checkout the `part4_transactions` git branch.{{%/sidenote%}}
with `make native`. The updated interface has two new flags - `balances` and `send`. As before,
usage instructions can be viewed by running `./cli.native -help`:

```
Interact with the blockchain

  cli.native SUBCOMMAND

=== subcommands ===

  balances  Print the balance of each user
  new       Create a new blockchain
  print     Print the blockchain
  send      Send coins from one user to another
  version   print version information
  help      explain a given subcommand (perhaps recursively)


```

First, let's create a new blockchain by running `./cli.native new`. As before, the data stored
in the blockchain can be viewed by running `./cli.native print`:

```json
{
  "blocks": [
    {
      "timestamp": 1515452242.8212757,
      "transactions": [
        [
          "Coinbase",
          { "outputs": [ { "value": 50.0, "recipient": "Satoshi" } ] }
        ]
      ],
      "prevhash": null,
      "nonce": 4153
    }
  ],
  "utxo": [ { "value": 50.0, "recipient": "Satoshi" } ]
}
```

Notice that I have hardcoded a coinbase transaction in the genesis block to
contain a single transaction output with a value of 50 and a recipient of `Satoshi` in homage to
the creator of bitcoin - Satoshi Nakamoto. Clearly, the balance of Satoshi is 50 coins. We can
check this by running `./cli.native balances`:

```
Satoshi: 50.000000
```

Let's send 20 coins from Satoshi to Alice by running 
`./cli.native send --sender Satoshi --recipient Alice --amount 20`. The updated balances can be
seen by running `./cli.native balances`:

```
Alice: 20.000000
Satoshi: 30.000000
```

As expected, Alice has 20 coins and Satoshi has 30 coins. Now let's see what happens when a user
has insufficient funds for a transaction. Let's have Alice send 40 coins to Bob with 
`./cli.native send --sender Alice --recipient Bob --amount 40`:

```
Uncaught exception:
  
  (Failure "Alice has insufficient funds")
```

Since Alice only has a balance of 20 coins, an exception is raised.

## Conclusion and next steps

We have completed the first step in building a cryptocurrency by using a blockchain as a secure
transaction ledger. We have implemented the data structures required for represent transactions
and modified the blockchain implementation to store them. We have also learned how to construct
valid transactions for sending coins between users which is helped by an elegant solution to
maintaining the UTXO set. However, we have not concretely defined the concept of *ownership* of
coins. What does it mean for Alice to have 20 coins? Currently, there is nothing to stop someone
from writing any transaction they like to the blockchain. For example, Carol could construct a 
perfectly valid transaction sending herself some coins belonging to Bob without his permission.
What we need is a way requiring a user to authenticate themselves and to prove they have
ownership over transaction outputs. Fortunately cryptographers have solved this problem
with *public key cryptography*. So join me in the next article where we will put the "crypto" in
our cryptocurrency :)
