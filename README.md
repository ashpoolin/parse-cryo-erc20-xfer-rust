# parse-cryo-erc20-xfer-rust
A command line parser for fast processing of ethereum block files produced by Paradigm's Cryo tool.

## Build
`cargo build --release` # ignore warnings, I'm v lazy

## Execute
`./target/release/parse-cryo-erc20-xfer-rust <COMMA DELIMITED ERC20 TRANSFER DATA>`

## How To
1. First, set up an ethereum node (reth + lighthouse is good), and Cryo
2. Get your transaction files from Cryo as follows:
```
    cryo transactions --blocks <START>:<END> --rpc http://localhost:8545 --max-concurrent-requests 20 --csv
```
3. You now have files that look like this:
```
    ethereum__transactions__18029000_to_18029345.csv
    ethereum__transactions__18028000_to_18028999.csv
    ethereum__transactions__18027000_to_18027999.csv
    ethereum__transactions__18026000_to_18026999.csv
    ethereum__transactions__18025000_to_18025999.csv
```
4. These are comma-delimited files with a header that looks like this:
```
    block_number,transaction_index,transaction_hash,nonce,from_address,to_address,value,input,gas_limit,gas_price,transaction_type,max_priority_fee_per_gas,max_fee_per_gas,chain_id
```
5. grep or [better] csvgrep (`sudo apt-get install csvkit`) a file to filter for erc20-compliant transfers (using identifier `input` like `a9059cbb`), and optionally, a token contract address (`to_address`). Note the contract address is all lowercase by default from Cryo. Here we do it for USDT:
```
    csvgrep -c input -m "a9059cbb" -z 1000000000 ethereum__transactions__18025000_to_18025999.csv | csvgrep -c to_address -m "0xdac17f958d2ee523a2206206994597c13d831ec7" -z 1000000000 > ethereum__transactions__18025000_to_18025999_erc20_raw.csv
```
6. next, we can call the parser directly from the command line. Example using for loop:
```
    echo "block,hash,source,destination,amount,contract" > ethereum__transactions__18025000_to_18025999_erc20_parsed.csv; \
    for line in `cat ethereum__transactions__18025000_to_18025999_erc20_raw.csv | tail -n +2`; \
    do \
        ./parse-cryo-erc20-xfer-rust $line >> ethereum__transactions__18025000_to_18025999_erc20_parsed.csv \
    done
```
OR, executed using GNU parallel:
```
    echo "block,hash,source,destination,amount,contract" > ethereum__transactions__18025000_to_18025999_erc20_parsed.csv; \
    cat ethereum__transactions__18025000_to_18025999_erc20_raw.csv | tail -n +2 | parallel -j 16 ./parse-cryo-erc20-xfer-rust {} >> ethereum__transactions__18025000_to_18025999_erc20_parsed.csv ; 
```

7. View it and or load it into a database:
```
    csvlook ethereum__transactions__18025000_to_18025999_erc20_parsed.csv
```

8. That's it! You now have a .csv file with the following columns. Note that the amount field is a very long integer, so should be treated as a string, until you convert to a numeric type (and correct for the number of `decimals`). Can do this in postgres like `select amount::decimal / 10^<# decimals> as uiamount`. You could create a postgres table and load it as follows: 
```
drop table if exists erc20_xfers_usdt;

create table erc20_xfers_usdt (
    block bigint,
    hash char(67),
    source char(43),
    destination char(43),
    amount character varying(255),
    contract char(43)
);

\copy erc20_xfers usdt from ethereum__transactions__18025000_to_18025999_erc20_parsed.csv csv header;
```

9. DONE
