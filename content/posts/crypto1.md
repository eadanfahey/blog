+++
draft = true
title = "Let's build a cryptocurrency (part 1): Blockchain"
date = "2017-10-20T15:46:53+01:00"
+++

The blockchain is the core data structure in most cryptocurrencies. While first popularised by
its use as a distributed ledger for bitcoin, the blockchain can be used for storing
arbitrary data - not just transaction records. There is an aura of mysticism surrounding the word
"blockchain" in the popular media, but in reality it is a rather simple data structure - a 
linked list of chunks of data called *blocks*. The real magic of the blockchain is not in the
data structure itself, but as a way of a maintaining a *decentralised* and *permissionless*
system for *consensus*.

In this article we will build a simple blockchain. The code is stored under the 
[`part1_blockchain`](https://github.com/eadanfahey/raicoin/tree/part1_blockchain) branch.

## Blocks

A block stores a chunk of arbitrary data. Bitcoin stores transactions in blocks, but there is no
reason why a block could not store any type of data - plain text, images, audio etc. A simple
block design consists of four components: *timestamp*, *data*, *previous block hash* and
the *nonce*.

{{% figure src="/images/block.png" caption="The contents of a block" %}}

We can implement a block as a `struct`, defining the `data` parameter to be a `String` for the
time being.

{{% marginnote %}}[`src/block.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/block.rs#L8){{% /marginnote %}}
{{< highlight rust >}}
#[derive(Serialize, Deserialize)]
pub struct Block {
    pub timestamp: i64,
    pub data: String,
    pub prev_block_hash: String,
    pub nonce: i64,
}
{{< /highlight >}}

The crate [`serde_json`](https://github.com/serde-rs/json) will come in handy for converting our
rust data structures to and from json strings. A nice feature of this crate is the `derive` macro
which automatically generates the `Serialize` and `Deserialize` traits for a data structure,
eliminating quite a bit of boilerplate code. The functions `serialize` and `deserialize`
defined below are simple wrappers around the `to_string` and `from_str` methods of `serde_json`,
respectively. They will be used extensively throughout the codebase.

{{% marginnote %}}[`src/serialize.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/serialize.rs){{% /marginnote %}}
{{< highlight rust >}}
pub fn serialize<T: serde::Serialize>(item: &T) -> String {
    serde_json::to_string(item).unwrap()
}

pub fn deserialize<'a, T: serde::Deserialize<'a>>(s: &'a str) -> T {
    serde_json::from_str(s).unwrap()
}
{{< /highlight >}}

### The Block Hash

To link blocks into a chain, and to ensure integrity of the data stored within a block, we need
a way of creating a creating a unique "fingerprint" to identify a block. This can be achieved
using *cryptographic hash function*. 

{{% figure src="/images/hashing.png" caption="Computing the hash of a block." %}}

A hash function takes some data of arbitrary size and produces a seemingly
random string of fixed length. This string serves as a unique{{% sidenote %}}In theory, it is possible for a "collision" to occur - a hash function producing the same hash for different inputs. In 2017, researchers at Google and the CWI research institute in Amsterdam [generated a collision](https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html) with the SHA-1 hashing algorithm. However, the computation required was immense - 6,500 years of CPU time alongside 110 years of GPU time!{{% /sidenote %}}
fingerprint for the data. Like bitcoin, we will use the SHA-256 hash function. The python
snippet below shows how changing the input to the hash function slightly produces a completely
different hash.

{{< highlight python >}}
hashlib.sha256(b"To be, or not to be").hexdigest()
#=> 80ed7fc26bc0bbb12ef93ae6dbc4bdaae8262e83e65163626e57c24b8fc316a8

hashlib.sha256(b"To be, or not to be.").hexdigest()
#=> 4a0aa387041df524f9acd333faecf2d1470db8b567473364186115bfc192f9f3
{{< /highlight >}}

Another amazing property of cryptographic hash functions is that they are *one-way* - it is easy to
compute the hash of some data, but given a hash it is computationally impractical to find the
original data. This feature will be important in the next article when we discuss *proof of work*.

The `hash` method defined below computes the hash of a `Block` by first serializing it to json,
and passing the resulting string through the `Sha256` hasher.

{{% marginnote %}}[`src/block.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/block.rs#L17){{% /marginnote %}}
{{< highlight rust >}}
impl Block {
    pub fn hash(&self) -> String {
        let mut hash = Sha256::new();
        hash.input_str(&serialize(&self));

        hash.result_str()
    }
    ...
}
{{< /highlight >}}

### Creating a block

With bitcoin, the process of creating a new block is called *mining*. The code below implements the
`mine` method, taking the `data` and `prev_block_hash` as parameters and returning a new `Block`.
The `nonce` will play it's role in the mining process when we implement proof of work. For now we
will just set it to `0`.

{{% marginnote %}}[`src/block.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/block.rs#L24)<br><br>{{% /marginnote %}}
{{< highlight rust >}}
impl Block {
    ...

    pub fn mine(data: String, prev_block_hash: String) -> Block {
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;

        let block = Block {
            timestamp,
            data,
            prev_block_hash,
            nonce: 0,
        };

        block
    }
}
{{< /highlight >}}

## The Blockchain

As mentioned previously, the blockchain is simply a series of blocks connected by pointing to the
previous block in the chain through the `prev_block_hash` parameter.

{{% marginnote %}}The chain is formed by referring to the hash of the previous block. For example, the hash of the first block is `a9f26` and the second block forms a link by referencing this hash in its `prev_block_hash` parameter.{{% /marginnote %}}
{{% figure src="/images/blockchain.png" %}}

Under the hood, we will implement the `Blockchain` as a map{{%sidenote%}} i.e. a `dict` for the pythonistas in the audience. {{%/sidenote%}}
from a block hash to its associated `Block`. The `last_block_hash` parameter will keep track of the
last block in the chain.

{{% marginnote %}}[`src/blockchain.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/blockchain.rs#L5){{% /marginnote %}}
{{< highlight rust >}}
#[derive(Serialize, Deserialize)]
pub struct Blockchain {
    blocks: HashMap<String, Block>,
    pub last_block_hash: String,
}
{{< /highlight >}}

### Retrieving and adding blocks

When adding a block to the chain we must first check if the block is valid, i.e. the
`prev_block_hash` of the new `Block` points to the `last_block_hash` of the `Blockchain`. If the new
block is valid we update the `last_block_hash` to the hash of the new block and insert the block
into the hash map. The `get_block` method is self-explanatory.

{{% marginnote %}}[`src/blockchain.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/blockchain.rs#L12){{% /marginnote %}}
{{< highlight rust >}}
impl Blockchain {
    pub fn get_block(&self, hash: &str) -> Option<&Block> {
        self.blocks.get(hash)
    }

    fn validate_block(&self, block: &Block) {
        if block.prev_block_hash != self.last_block_hash {
            panic!("Error: invalid previous_block_hash")
        }
    }

    pub fn add_block(&mut self, block: Block) {
        self.validate_block(&block);
        self.last_block_hash = block.hash();
        self.blocks.insert(block.hash(), block);
    }
    ...
}
{{< /highlight >}}

### Creating a new blockchain

A new blockchain starts off with a single block called the `genesis block`{{%sidenote%}}The genesis block of the bitcoin blockchain can be viewed [here](https://blockchain.info/block/000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f).{{%/sidenote%}}.
Since there are no previous blocks in the chain, the `prev_block_hash` of the genesis block is set
to the empty string.

{{% marginnote %}}[`src/blockchain.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/blockchain.rs#L28){{% /marginnote %}}
{{< highlight rust >}}
impl Blockchain {
    ...

    pub fn new() -> Blockchain {
        let mut blockchain = Blockchain {
            blocks: HashMap::new(),
            last_block_hash: String::new(),
        };

        let data = "Genesis block".to_owned();
        let prev_block_hash = "".to_owned();
        let genesis = Block::mine(data, prev_block_hash);
        blockchain.add_block(genesis);

        blockchain
    }
    ...
}
{{< /highlight >}}

## Wrapping things up

We have finished implementing the basic functionality
{{%sidenote%}}I have not explained the [`iter`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/blockchain.rs#L42) method of `Blockchain`. It simply iterates through the blockchain, starting from the last block inserted.{{%/sidenote%}}
of the blockchain. Let's test it out by
creating a new blockchain and adding some blocks to it.

{{% marginnote %}}[`src/main.rs`](https://github.com/eadanfahey/raicoin/blob/part1_blockchain/src/main.rs){{% /marginnote %}}
{{< highlight rust >}}
fn main() {

    let mut blockchain = Blockchain::new();

    let block1 = Block::mine(
        "Infinte Jest".to_owned(),
        blockchain.last_block_hash.clone(),
    );
    blockchain.add_block(block1);

    let block2 = Block::mine(
        "Gravity's Rainbow".to_owned(),
        blockchain.last_block_hash.clone(),
    );
    blockchain.add_block(block2);

    for (hash, block) in blockchain.iter() {
        println!("==============================\n");
        println!("hash: {}\ncontents: {}\n", hash, block);
    }
}
{{< /highlight >}}

Compile and run the code with `cargo run` and you should see something like:

```
==============================

hash: 520ebe8a3e1da18adb7a2cf5c1ecfeab3be0274218f0cc2f19bab1b73b5f567d
contents: {
  "timestamp": 1508598594,
  "data": "Gravity's Rainbow",
  "prev_block_hash": "99c16ef08e7c9e8905bc27917bc956565efb8c8a646f6da7761eeaf9de072c81",
  "nonce": 0
}

==============================

hash: 99c16ef08e7c9e8905bc27917bc956565efb8c8a646f6da7761eeaf9de072c81
contents: {
  "timestamp": 1508598594,
  "data": "Infinte Jest",
  "prev_block_hash": "16ae3c6c1acaf6eb991dbf320d07f6b8e65cee51a4f4d4ddce06df86951c1b28",
  "nonce": 0
}

==============================

hash: 16ae3c6c1acaf6eb991dbf320d07f6b8e65cee51a4f4d4ddce06df86951c1b28
contents: {
  "timestamp": 1508598594,
  "data": "Genesis block",
  "prev_block_hash": "",
  "nonce": 0
}
```

Nice! As you can see we begin iterating from the end of the blockchain with the `prev_block_hash`
matching the the hash of the previous block in the chain.

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
from "Catch-22" to "Atlas Shrugged". In doing so, she changes the hash of `Block 1` from `3rtx7` to
`rf81a`. But, the previous block hash of `Block 2` still points to the old hash - `3rtx7` - and so,
the blockchain has been split in two.

{{% figure src="/images/carol-invalid-blockchain.png" caption="Carol has split her blockchain in two!"%}}

Carol wants to convince Bob that Atlas Shrugged really is one of Alice's favourite books. She has 
two options:

  1. Tell Bob that the lower portion is the true state of the world, pretending that the latter
     portion of the blockchain never existed.
  2. Somehow, join the the two chains back together.
  
The first option is unlikely to pan out in a decentralised network. Like all good blockchain users
Bob is cautious. When downloading the blockchain he doesn't just take Carol's word, but also asks
other nodes in the network about their copy of the blockchain. On discovering that others have a
longer blockchain, there is no reason for Bob to take Carol's blockchain as the truth.

To deceive Bob, Alice must go for option two - join the two chains back together.

Carol to updates the previous block hash of `Block 2` to point to the hash of the modified `Block 1`.
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
