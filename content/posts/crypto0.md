+++
title = "Let's build a cryptocurrency (part 0): Introduction"
draft = true
date = "2017-10-19T20:13:17+01:00"
+++

{{% blockquote footer="Richard Feynman" %}}
What I cannot create, I do not understand.
{{% /blockquote %}}

Programming is an amazing medium for learning. Encoding ones thought process in a program requires
a thorough comprehension of the underlying theory. In the upcoming series of articles we will build
the core of a basic cryptocurrency, modeled on bitcoin, using
[Rust](https://www.rust-lang.org/en-US/). Each article in this series is shown below:

  * [ Part 1: Blockchain ](../crypto1)
  * [ Part 2: Proof of Work ](../crypto2)
  * [ Part 3: Persistence and a CLI ](../crypto3)
  * [ Part 4: Transactions ](../crypto4)
  * [ Part 5: Cryptography ](../crypto5)
  * [ Part 6: Mining Rewards ](../crypto6)

Starting with the basic data structure - the blockchain - we will iteratively flesh out our
cryptocurrency, learning how the blockchain is secured, how the blockchain represents a public
ledger of transactions and how public key cryptography is used to authenticate transactions. The
powerful type system of Rust is great for helping ensure correctness and making the refactoring
process a breeze. I will assume you have some prior experience with Rust.
{{% sidenote "sn-example" %}}I recommend reading the official Rust [ book ](https://doc.rust-lang.org/stable/book/) to get started.{{% /sidenote %}}
At the beginning, I will make extensive use of `unwrap()` and `panic!` in place of proper error
handling. Towards the end we will refactor our code to make proper use of Rust's elegant error
handling built into its type system.

All cryptocurrencies need a catchy name. We will call ours *raicoin* in reference to the
[ Rai stone ](https://en.wikipedia.org/wiki/Rai_stones)
currency of the Micronesian islands.
{{% figure type="margin" src="/images/rai_stone.jpg" caption="The largest Rai stones can weigh up to 4 tonnes!" %}}
