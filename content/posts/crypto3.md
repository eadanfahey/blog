+++
title = "Let's build a cryptocurrency (part 3): Persistence and a CLI"
date = "2017-12-23"
+++

In the [previous article](../crypto2) we secured the blockchain by adding the proof-of-work
requirement to mining a block. Before we learn how to build
a cryptocurrency atop a blockchain, I first want to add some functionality to make it easier
to interact with the blockchain. In this article we will build a commandline interface to the
blockchain and the ability the persist the state of the blockchain to file.

The code for this article is stored under the
[`part3_cli`](https://github.com/eadanfahey/ORaiml/tree/part3_cli) git branch.
The updated codebase can be compared to the the code from part 2
[here](https://github.com/eadanfahey/ORaiml/compare/part2_pow...part3_cli).

## Persistence

Currently, we start with a new blockchain each time we run the program. The functions
defined below - `to_file` and `from_file` - allow us to save a blockchain to a
text file in the JSON format and read a blockchain from a file respectively.

{{% marginnote %}}[`src/blockchain.ml`](https://github.com/eadanfahey/ORaiml/blob/part3_cli/src/blockchain.ml#L44){{% /marginnote %}}
{{< highlight ocaml >}}
let to_file t filename =
  let data = to_yojson t |> Yojson.Safe.to_string in
  Out_channel.write_all filename ~data:data


let from_file filename =
  let data = In_channel.read_all filename in
  match Yojson.Safe.from_string data |> of_yojson with
  | Ok bc -> bc
  | Error e -> failwith e
{{< /highlight >}}


## Commandline interface

The commandline interface to the blockchain is implemented in 
[`cli.ml`](https://github.com/eadanfahey/ORaiml/blob/part3_cli/src/cli.ml). The code itself is
not very interesting so I won't go through it here. As usual, run `make native` to compile the
code to a native executable. The CLI help can be viewed by running `./cli.native -help`:
```
Interact with the blockchain

  cli.native SUBCOMMAND

=== subcommands ===

  add-block  Add a block to the blockchain
  new        Create a new blockchain
  print      Print the blockchain
  version    print version information
  help       explain a given subcommand (perhaps recursively)
```

Let's test it out by making a new blockchain and adding a few blocks to it.

{{< highlight bash >}}
./cli.native new
./cli.native add-block --data "Doctor Zhivago"
./cli.native add-block --data "Down and out in Paris and London"
./cli.native print
{{< /highlight >}}

Running the above should produce an output similar to:

```
Added block with hash 0009e1ab149b424b3f490ae19d97b15f7856fbe244621a68deeb2bb4872a441f

Added block with hash 0000fa0f37ac132e7217fef8021d5e6cf4383132ece2768bd2d93294ece9c75d

[
  {
    "timestamp": 1514067328.7027342,
    "data": "Down and Out in Paris and London",
    "prevhash":
      "0009e1ab149b424b3f490ae19d97b15f7856fbe244621a68deeb2bb4872a441f",
    "nonce": 6538
  },
  {
    "timestamp": 1514067328.6273808,
    "data": "Doctor Zhivago",
    "prevhash": null,
    "nonce": 7898
  }
]
```

## Conclusion

This article was an interlude on our journey to creating a cryptocurrency. Now that we have a nice
CLI for our blockchain there are exciting times ahead. In the next article we will begin
discussing how to build a transaction ledger atop our blockchain implementation.
