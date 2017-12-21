+++
title = "Let's build a cryptocurrency (part 0): Introduction"
date = "2017-12-20"
+++

{{% blockquote footer="Richard Feynman" %}}
What I cannot create, I do not understand.
{{% /blockquote %}}

Welcome to the introductory article to a series titled "Let's build a cryptocurrency". Over the
course of the series we will build a toy cryptocurrency together using the programming language
[OCaml](https://ocaml.org/).

  * [ Part 1: Blockchain ](../crypto1)
  * [ Part 2: Proof of Work ](.) (coming soon!)
  * [ Part 3: Persistence and a CLI ](.) (coming soon!)
  * [ Part 4: Transactions ](.) (coming soon!)
  * [ Part 5: Cryptography ](.) (coming soon!)
  * [ Part 6: Mining Rewards ](.) (coming soon!)

Starting with the basic data structure - the blockchain - we will iteratively flesh out our
cryptocurrency, learning how the blockchain is secured, how the blockchain represents a public
ledger of transactions and how public key cryptography is used to authenticate transactions.

## Why OCaml?

OCaml is a functional programming language with a strong type system and an emphasis on
immutable data structures. While not very popular, OCaml is a very elegant programming language
with [very good](https://benchmarksgame.alioth.debian.org/u64q/compare.php?lang=ocaml&lang2=java) 
performance and is used by some heavy-hitters in [industry](https://ocaml.org/learn/companies.html).
For newcomers to OCaml, I highly recommend the book "Real World OCaml" which can be read for free
[online](https://ocaml.org/learn/companies.html). If you just want to compile the code
to follow along with the series, this book has good
[instructions](https://github.com/realworldocaml/book/wiki/Installation-Instructions) 
for installing OCaml.

I have used OCaml 4.06.0 during this series. The libraries we will be using can be installed with
OCaml's package manager opam:

```
opam install core utop merlin ocp-indent cryptokit \
yojson ppx_deriving_yojson bignum
```

## What's with the name?

All cryptocurrencies need a catchy name, and I have called our toy one *ORaiml* in reference to
OCaml and the [Rai stone](https://en.wikipedia.org/wiki/Rai_stones) currency of the 
Micronesian islands.
{{% figure type="margin" src="/images/rai_stone.jpg" caption="The largest Rai stones can weigh up to 4 tonnes!" %}}
These Rai stones are cicular disks with a hole in the centre, which were carved by the Yapese
culture on the island of Yap. For hundreds of years these stones were
used as a currency for exchange of goods and a store of wealth. Amusingly, after Europeans started
arriving with their more advanced tools, their monetary system experienced huge inflation after
people began making larger and larger Rai stones. The Rai stone currency is a reminder that
currencies can take many forms - paper, metals, stone, bits on a bank's hard-drive or a globally
distributed, decentralised, secure and permissionless electronic ledger such as bitcoin.

To kick things off, the first article in the series will introduce the blockchain data structure
which you can read [here](../crypto1).
