# optimism-state-export

A gentle guide to prepare storage for new execution client for OP stack. Preliminary info: goerli bedrock transition was done at block 4061224.

TL;DR: Jump to [Game Plan](#game-plan)

## Historical data

To build new execution client for OP stack, we must import historical data of optimism. Lets break down what exactly the history data means. The historical data consists of below segments.

1. Block header
    - state root, transaction trie root, transaction receipt trie root is included
2. Transaction trie
    - Transactions are stored at leaves
3. Transaction receipt trie
    - Transaction receipts are stored at leaves
4. World state trie
    - Account states(balance, nonce, code, storage trie root) is stored at leaves

## Prebedrock Block Execution is Impossible

Simply thinking, you may claim that new client may start with genesis, and by only knowing the block header and the transaction trie, it can execute the transactions and advance the block state, contructing transaction receipt trie and world state trie.

However, **new client has no functionality of executing the pre-bedrock transactions**. This is because pre-bedrock state transition was only done via [l2geth](https://github.com/ethereum-optimism/optimism/tree/develop/l2geth), which does not have EVM equivalence. 

## World State Trie Dump to the Rescue

Therefore we need to extract the entire world state trie, starting from bedrock block(block number 4061224). As you can see in [etherscan](https://goerli-optimism.etherscan.io/block/4061224), this block does not contain transactions not to make ambiguity during bedrock transition. Bedrock supports EVM equivalence, so every transactions included in bedrock era can be natively executed in your new execution client, with some tweaks. The tweaks will be the diff introduced between geth and [op-geth](https://github.com/ethereum-optimism/op-geth). [Here](https://op-geth.optimism.io/) are the detailed diff explanation.

You may create the state dump using [public info](https://community.optimism.io/docs/developers/bedrock/public-testnets/#):
- [Bedrock Data Directory](https://storage.googleapis.com/oplabs-goerli-data/goerli-bedrock.tar)
    - `7.5G`. This tarball contains the following info
        - Block header until block 4061224
        - Transaction until block 4061224
        - Transaction receipt until block 4061224
        - World state trie **only for** genesis and 4061224
- [Legacy Geth Data Directory](https://storage.googleapis.com/oplabs-goerli-data/goerli-legacy-archival.tar)
    - `62G`. This tarball contains the following info
        - Block header until block 4061223
        - Transaction until block 4061223
        - Transaction receipt until block 4061223
        - **Every** world state trie until block 4061223

By using upper info, you must construct file which contains state trie to reconstruct world state trie for block 4061224, starting from bedrock. 

Luckily, here is the `1.39G` json file which captures every info for recontructing world state trie for block 4061224. [alloc_everything_4061224_final.json](https://drive.google.com/file/d/1k9yopW6F8SyHAR-8JT2hfxptQGT-DqKe/view?usp=sharing). It was super painful to derive all the preimages! In theory, every data to create this world state trie is included in bedrock data directory. However, we had some problems with state dump, so we also used legacy geth data directory to create the full world state trie. The given json file follows the [genesis file structure](https://arvanaghi.com/blog/explaining-the-genesis-block-in-ethereum/). Detailed steps to create this json will be added in this repo in the future.

## Data Export

All the data export artifacts will be shared asap. You do not have to follow steps. You can use files that we provided in the [game plan](#game-plan). We followed below steps because op-geth dump functionality is not working at the moment.

1. Initialize your new OP stack execution engine's storage with the [optimism-goerli genesis file](https://github.com/testinprod-io/erigon/blob/pcw109550/state-import/state-import/genesis.json).
2. Dump block headers and transactions from block number 1 to 4061224 from the archive. This can be done via l2geth's `export` command. RLP encoded file will be created.
3. Dump transaction receipts from block 1 to 4061224 from the archive. I wrote the custom geth command `export-receipts` to do this. See [here](https://github.com/testinprod-io/optimism/commit/6c675653e5db865415d3260fce50f3d5c39c267e) for details. RLP encoded file will be created.
4. Dump world state trie at block 4061224. [alloc_everything_4061224_final.json](https://drive.google.com/file/d/1k9yopW6F8SyHAR-8JT2hfxptQGT-DqKe/view?usp=sharing)

## Game Plan

1. Import [optimism-goerli genesis file](https://github.com/testinprod-io/erigon/blob/pcw109550/state-import/state-import/genesis.json) to your new client
2. Import block headers and transactions to your new client.
     - RLP encoded block: [export_0_4061224](https://drive.google.com/file/d/1z1pGEhy8acPi_U-6Sz0oo_-zJSzU8zb-/view?usp=sharing)
3. Import transaction receipts to your new client
     - RLP encoded receipt: [export_receipt_0_4061223](https://drive.google.com/file/d/1QJpv-SNv6I3j9z4FfHzZ3fHlCuFMn8b0/view?usp=sharing)
     - There is no receipt for block 4061224 because no transaction at block 4061224.
4. Import world state trie at block 4061224: [alloc_everything_4061224_final.json](https://drive.google.com/file/d/1k9yopW6F8SyHAR-8JT2hfxptQGT-DqKe/view?usp=sharing) to your new client. I wrote [custom import functionality](https://github.com/testinprod-io/erigon/blob/pcw109550/state-import/turbo/app/import.go) for op-erigon to achieve this.

You may ask that is it okay not to have world state trie for prebedrock block. You may simply relay the requests to l2geth node. Daisy chain will handle these prebedrock jobs.

## Wait, How can I trust your huge json?

Here is the huge json: [alloc_everything_4061224_final.json](https://drive.google.com/file/d/1k9yopW6F8SyHAR-8JT2hfxptQGT-DqKe/view?usp=sharing). You can check the validity of this json by recontructing the world state trie, and check the merkle root. Query block 4061224, get the world state trie's root included at block header and compare it by yourself. You may refer to the [creation/validation script](https://github.com/testinprod-io/optimism/blob/pcw109550/l2geth%400.5.31/l2geth/parse/create_alloc.py) written by me.
