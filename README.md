# Bitcoin Chainstate Parser

This simple Ruby script reads the Bitcoin `~/.bitcoin/chainstate` LevelDB database and prints out each UTXO entry.

I wrote this script so that I could understand how the data inside the database had been compressed. It's not very fast, but it's good for educational purposes.

**NOTE: If you're looking to get the entire UTXO set from the chainstate database quickly, check out this tool instead: [http://github.com/in3rsha/bitcoin-utxo-dump](bitcoin-utxo-dump). I basically wrote this Ruby script so that I could get an understanding of the chainstate database structure before writing the faster Go tool.**

## Usage

You will need `libleveldb-dev` and the `leveldb-ruby` gem:

```
sudo apt install libleveldb-dev
sudo gem install leveldb-ruby
```

Then you can just:

```
ruby chainstate.rb
```

## What is the chainstate database?

The `~/.bitcoin/chainstate` folder contains a list of every unspent transaction output ([UTXO](http://learnmeabitcoin.com/glossary/utxo)) in the blockchain. These UTXOs are stored in a [LevelDB](http://leveldb.org/) database, which is a key-value store database (like Redis), but it uses flat files instead of a database server.

This database allows bitcoin to get **fast access to unspent outputs**. This is vitally important for speeding up transaction validation, as bitcoin needs to grab the "locking scripts" for each output being spent in a transaction during validation. Without this database, bitcoin would need to trawl through the entire blockchain to find each output.

Now, seeing as this chainstate database will inevitably contain a large number of outputs, **the data has been compressed** as much as possible. So whilst this means the data takes up as little disk space as possible, it also means that it's far from human-readable.

Furthermore, the data inside the database has also been _obfuscated_ to help prevent triggering any anti-virus software on some computers (although I'm not sure why that happens).

## How are the UTXOs stored in LevelDB?

LevelDB is a key:value store, so you could think of the UTXO database as looking something like this:

```
key1:value
key2:value
key3:value
...
```

### Keys

The keys for UTXOs in the database have the following structure:

```
key:   43000006b4e26afc5d904f239930611606a97e730727b40d1d82d4f3f1438cf2a101
       <><--------------------------------------------------------------><>
       /                              |                                   \
   type                             txid                                   vout
```

 * The **first byte** indicicates the type of entry. A UTXO entry starts with `0x43`, which is "**C**" in ASCII.
 * The **next 32 bytes** is the [TXID](http://learnmeabitcoin.com/glossary/txid) for the transaction the output was created in. This is in _[little-endian](http://learnmeabitcoin.com/glossary/little-endian)_, so you will need to swap the byte order if you want to search for it on the blockchain.
 * The **last byte** is the [VOUT](http://learnmeabitcoin.com/glossary/vout), which is the index number for an output in a transaction.

### Values

First of all, every value in the database has been obfuscated, so you will need the get the `obfuscate_key` (which is also the first entry in the database):

```
obfuscate_key = chainstate.get ["0e006f62667573636174655f6b6579"].pack("H*") # "\x0E\x00obfuscate_key"
#=> b12dcefd8f872536
```

Now, an obfuscated `value` from the database will look something like this:

```
71a9e87d62de25953e189f706bcf59263f15de1bf6c893bda9b045
```

To deobfuscate it, you just need to extend the `obfuscate_key` to the same length of the value, and then **XOR** the value with the extended `obfuscate_key`:

```
71a9e87d62de25953e189f706bcf59263f15de1bf6c893bda9b045  <- value
b12dcefd8f872536b12dcefd8f872536b12dcefd8f872536b12dce  <- extended obfuscate_key
c0842680ed5900a38f35518de4487c108e3810e6794fb68b189d8b  <- deobfuscated value (XOR)
```

**NOTE:** The Bitwise XOR operator is useful for toggling bits on and off (from 0 to 1 and vice versa). Therefore, XORing the value with the key obfuscates it, and XORing it again with the same key will de-obfuscate it.

The deobfuscated `value` of a UTXO entry in the database has the following structure:

```
value:  c0842680ed5900a38f35518de4487c108e3810e6794fb68b189d8b
        <----><----><><-------------------------------------->
          /      |    \                   |
    varint    varint   varint          script <- hash160 for P2PKH and P2SH, public key for P2PK, full script for rest
        |    (amount)  (nSize)
        |        \
 decode |         decompress
        |
        |
   100000100001010100110
   <------------------>^coinbase
          height
```

To read through the data, you need to be able to read and decode [**varints**](https://developers.google.com/protocol-buffers/docs/encoding#varints). These are just numbers that have been serialized in a specific way to help reduce the amount of space they take up within structured data. They are popular in binary protocols (because they are more efficient than fixed-length fields when transmitting numbers that vary in length).

#### Varints

To read a varint, you just need to keep reading bytes until the **first bit is not set**. For that we need to look at each byte in its _binary representation_. So using the example above:

```
c0842680ed5900a38f35518de4487c108e3810e6794fb68b189d8b

c0 = 11000000
84 = 10000100
26 = 00100110 <- first bit not set, so stop reading bytes
```

So the first varint we have read is `c08426`. To decode this varint in to an actual value, you just take the last 7 bits of each of these bytes, add 1 to each (except for the last), then concatenate them.

```
11000000 10000100 00100110
 1000000  0000100  0100110 <- last 7 bits of each byte
 1000001  0000101  0100110 <- add 1 to each byte (except for the last)
 100000100001010100110     <- concatenate
```

So there we have read and decoded a varint.

#### 1. First Varint

The first varint in the `value` gives you the **height of the block** the transaction output was included in (every bit except for the last), as well as whether the output is from a **coinbase transaction** or not (the last bit).

```
value:  c0842680ed5900a38f35518de4487c108e3810e6794fb68b189d8b
        <---->
          /
      varint
        |
        | decode
        |
   100000100001010100110
   <------------------>^coinbase
          height
```

So in this example:

 * **Height** = `0b1110100000011111011` = `532819`
 * **Coinbase** = `0b0` = not a coinbase

#### 2. Second Varint

The second varint you get from the `value` gives you the `amount` of bitcoins stored in the output (in Satoshis).

```
value:  c0842680ed5900a38f35518de4487c108e3810e6794fb68b189d8b
              <---->
                |
              amount (compressed)
```

However, this value has been **compressed** in a specific way, so you need to decompress it to get the actual value. I'm not entirely sure how this works, but the process can be found in the `decompress_value.rb` function (or in the [bitcoin source code](https://github.com/bitcoin/bitcoin/blob/master/src/compressor.cpp#L141)).

Nonetheless, the result of decoding the varint above and decompressing it looks like this:

```
80ed59  <- varint  (hexadecimal)
30553   <- varint decoded (decimal)
339500  <- decompressed (decimal)
```

#### 3. Third Varint

The third and final varint is referred to as `nSize`. This essentially indicates the _type_ of upcoming [locking script](http://learnmeabitcoin.com/glossary/scriptPubKey).

```
value:  c0842680ed5900a38f35518de4487c108e3810e6794fb68b189d8b
                    <>
                     \
                      nSize
```

The values for `nSize` indicate the following about the upcoming script data:

```
00  = P2PKH <- upcoming script is the hash160 public key
01  = P2SH  <- upcoming script is the hash160 of a script
02  = P2PK  <- upcoming script is a compressed public key (nsize makes up part of the public key) [y=even]
03  = P2PK  <- upcoming script is a compressed public key (nsize makes up part of the public key) [y=odd]
04  = P2PK  <- upcoming script is an uncompressed public key (but has been compressed for leveldb) [y=even]
05  = P2PK  <- upcoming script is an uncompressed public key (but has been compressed for leveldb) [y=odd]
06+ =       <- indicates thes size of the upcoming script (subtract 6 to get the actual size in bytes)
```

The script data has been compressed as much as possible. For example, the [`P2PKH`](http://learnmeabitcoin.com/glossary/p2pkh), [`P2SH`](http://learnmeabitcoin.com/glossary/p2sh), and [`P2PK`](http://learnmeabitcoin.com/glossary/p2pk) scripts follow the same pattern of OP_CODES, so the only part of these scripts that's unique is the public keys and script hashes, so we just store those in LevelDB instead of the full script.

Other non-standard scripts get stored in full, so the `nSize` is just used the indicate the size of those scripts (subtract 6 from this value thought to get the script size, to account for the fact that the values of 0-5 were taken for specifying a specific script type).

#### 4. Remaining Data

The remainding data is the `script data`. This is going to be a **public key hash**, **script hash**, **public key**, or a **full script**.

For example, the `nSize` was `00`, which means that this is a **hash160 public key** from inside a `P2PKH` script:

```
value:  c0842680ed5900a38f35518de4487c108e3810e6794fb68b189d8b
                      <-------------------------------------->
                                          |
                                        script <- hash160 for P2PKH and P2SH, public key for P2PK, full script for rest
```

**TIP:** You can get the **address** from this script data if it's a `P2PKH`, `P2SH`, `P2WPKH`, or `P2WSH` script.

## Chainstate Parsers

  * [github.com/in3rsha/bitcoin-utxo-dump](https://github.com/in3rsha/bitcoin-utxo-dump)
  * [github.com/sr-gi/bitcoin_tools](https://github.com/sr-gi/bitcoin_tools)
  * [github.com/mycroft/chainstate](https://github.com/mycroft/chainstate)

## Links

  * <https://github.com/bitcoin/bitcoin/blob/master/src/compressor.cpp>
  * <https://bitcoin.stackexchange.com/questions/61907/uxto-db-structure>

## Thanks

  * The writing of this script was helped massively by the [bitcoin_tools](https://github.com/sr-gi/bitcoin_tools) repo from [Sergi Delgado Segura](https://github.com/sr-gi).
