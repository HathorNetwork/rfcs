- Feature Name: private_network_guide
- Status: Draft
- Start Date: 2021-09-27
- RFC PR: (leave this empty)
- Hathor Issue: (leave this empty)
- Author: Luis Helder <luislhl@hathor.network>

# Summary

This guide will show you how to run a local private Hathor network, with its own full-nodes, miner, wallets and explorer.

By the end of the guide you will have a fully working private network with the following setup:
- 4 hathor-core full-nodes connected with each other, each dedicated to a different purpose
- A cpu-miner mining blocks in the network
- A tx-mining-service to be used by Wallets to mine transactions
- 2 instances of hathor-wallet-headless configured with 2 different wallets

# Running the full-nodes

The first step when creating a new network is to create its genesis output script, and its three initial `txs`: 1 block that will include the genesis output script and 2 empty transactions. To learn more about blocks and transactions, check [this RFC](./0015-anatomy-of-tx.md).

We will need a valid address to be used when generationg the output script.

So this is what we will do first.

## Generating an address for the genesis

We will use our hathor-wallet-headless to generate a new wallet seed. Keep this seed safe if you are configuring a network for a real use case.


```sh
export SEED=$(docker run --entrypoint npm hathornetwork/hathor-wallet-headless run generate_words | tail -1)
echo "Your seed is '$SEED'"
echo "Keep this seed, we will use in the end of the guide."
```

Then, start a hathor-wallet-headless server:

```sh
cat << EOF > .env
HEADLESS_HTTP_PORT=8000
HEADLESS_NETWORK=testnet
HEADLESS_SERVER=https://node1.testnet.hathor.network/v1a/
HEADLESS_SEED_DEFAULT=default
EOF

docker run -itd --name wallet-headless --network="host" --env-file=.env hathornetwork/hathor-wallet-headless
sleep 10
```

Initialize the wallet using your generated seed and get an address for it. Note that we are replacing the previously exported variable `SEED` here.

```sh
curl -X POST --data "wallet-id=wallet" --data "seed=$SEED" http://localhost:8000/start
sleep 5
curl -H "X-Wallet-Id: wallet" http://localhost:8000/wallet/address
```

Write down this address, we will use it in the next section.

Finally, stop the hathor-wallet-headless.

```sh
docker stop wallet-headless
docker rm wallet-headless
```

### Important Note

Note that we used a server from our public testnet in the config above. This was done because the hathor-wallet-headless needs it to be able to start a wallet and generate an address for it in the following commands. From now on, we won't need to do it anymore.

However, to make things easier, we use the same address prefix as the public testnet in the next sections when configuring our private network, since we used it to generate an address. Otherwise, we would need to make an address translation from one prefix to another.

## Creating the genesis block and transactions
Now that we have an address, we will use it to create our genesis block and our two initial transactions.

Run this code in a python shell inside our hathor-core Docker image, to create the output script for the genesis. Keep note of the returned value to be used in the next steps as the GENESIS_OUTPUT_SCRIPT.

You need to replace `<address>` with the address you got in previous steps.

```sh
docker run -it --entrypoint python  hathornetwork/hathor-core -c "
from hathor.transaction.scripts import P2PKH
from hathor.crypto.util import decode_address, get_address_b58_from_bytes, get_checksum

testnet_address = decode_address('<address>')

privatenet_address = b'\x70' + testnet_address[1:]
privatenet_address = privatenet_address[:-4] + get_checksum(privatenet_address[:-4])

output = P2PKH.create_output_script(address=privatenet_address).hex()
print('GENESIS_OUTPUT_SCRIPT:', output)
print('privatenet_address:', get_address_b58_from_bytes(privatenet_address))
"
```

From now on, everytime you need to input some address, you have to use the `privanet_address` we generated here.

Now to configure the parameter of our new network, we create a new file in `~/hathor-private-tutorial/conf/privnet.py`, with the following content. Note that we need to replace the `GENESIS_OUTPUT_SCRIPT` variable from the previous command.

```sh
mkdir -p ~/hathor-private-tutorial/conf

cat << EOF > ~/hathor-private-tutorial/conf/privnet.py
from hathor.conf.settings import HathorSettings

SETTINGS = HathorSettings(
    P2PKH_VERSION_BYTE=b'\x70',
    MULTISIG_VERSION_BYTE=b'\x93',
    NETWORK_NAME='privatenet-tutorial',
    BOOTSTRAP_DNS=[],
    # Genesis stuff
    GENESIS_OUTPUT_SCRIPT=bytes.fromhex("<GENESIS_OUTPUT_SCRIPT>"),
    GENESIS_TIMESTAMP=1632426140,
    MIN_TX_WEIGHT_K=0,
    MIN_TX_WEIGHT_COEFFICIENT=0,
    MIN_TX_WEIGHT=8,
    REWARD_SPEND_MIN_BLOCKS=20
)
EOF
```

Run the python shell again, with additional parameters to use our newly created settings file, and passing a script to mine the genesis block and 2 initial transactions. Keep note of the printed outputs, they will be used next.

```sh
docker run -it --entrypoint python  -v ~/hathor-private-tutorial/conf:/privnet/conf --env HATHOR_CONFIG_FILE=privnet.conf.privnet hathornetwork/hathor-core -c "
from hathor.conf import HathorSettings
from hathor.transaction import genesis

settings = HathorSettings()

# This should output 'privatenet-tutorial'
print('NETWORK', settings.NETWORK_NAME)

for tx_name in ['BLOCK_GENESIS', 'TX_GENESIS1', 'TX_GENESIS2']:
    tx = getattr(genesis, tx_name)
    tx_hash = tx.start_mining(update_time=False).hex()
    print(f'{tx_name}_HASH', tx_hash)
    print(f'{tx_name}_NONCE', tx.nonce)
"
```

Include the following block in the file `~/hathor-private-tutorial/conf/privnet.py`, replacing with the outputed values from the previous command. Beware the names outputed and the names of the variables are slightly different, but it should be clear which is which.

```py
    GENESIS_BLOCK_NONCE=,
    GENESIS_BLOCK_HASH=bytes.fromhex(''),
    GENESIS_TX1_NONCE=,
    GENESIS_TX1_HASH=bytes.fromhex(''),
    GENESIS_TX2_NONCE=,
    GENESIS_TX2_HASH=bytes.fromhex(''),
```

This concludes the configuration of our new network. Next we will use this configuration to run our full-nodes.

If you want to know more about additional options to customize your network configuration, we will have an additional section about this in the end of the guide.


## Running our full-nodes
We will run 4 full-nodes. Each one will have a role our setup, and we will name them accoding to that role.

We will setup a simple network where we have a central fullnode to which all other fullnodes connect to form the P2P network. However, you could have whatever configuration you need in more complex scenarios.

### Fullnode Main
Our first full-node will be named `fullnode-main`, because it will be our central fullnode where other fullnodes connect to start syncing between them.

```sh
mkdir -p ~/hathor-private-tutorial/fullnode-main

docker run -ti --network="host" -v ~/hathor-private-tutorial/fullnode-main:/data:consistent -v ~/hathor-private-tutorial/conf:/privnet/conf --env PYTHON_PATH=/privnet/conf --env HATHOR_CONFIG_FILE=privnet.conf.privnet hathornetwork/hathor-core run_node --cache --status 8080 --listen tcp:40403 --data /data --x-fast-init-beta --wallet-index
```

This command configures the fullnode to:
- Listen for P2P connections on port 40403
- Expose its HTTP api on port 8080
- Use `~/hathor-private-tutorial/fullnode-main` as its data directory in the host.
- Use the configuration file we create for the network in `hathor.conf.privnet`
- Enable `wallet index`, which is needed to perform some kinds of operations in the api, like getting transactions information.

### Fullnode Wallet 1
This fullnode will be used by one of our `hathor-wallet-headless` instances, that's we are calling it `fullnode-wallet-1`.

```sh
mkdir -p ~/hathor-private-tutorial/fullnode-wallet-1

docker run -ti --network="host" -v ~/hathor-private-tutorial/fullnode-wallet-1:/data:consistent -v ~/hathor-private-tutorial/conf:/privnet/conf --env PYTHON_PATH=/privnet/conf --env HATHOR_CONFIG_FILE=privnet.conf.privnet hathornetwork/hathor-core run_node --cache --status 8081 --data /data --x-fast-init-beta --wallet-index --bootstrap tcp://127.0.0.1:40403
```

This command configures the fullnode to:
- Expose its HTTP api on port 8081
- Connect to `tcp://127.0.0.1:40403` (fullnode-main) to sync with the P2P network
- Use `~/hathor-private-tutorial/fullnode-wallet-1` as its data directory in the host.
- Use the configuration file we create for the network in `hathor.conf.privnet`
- Enable `wallet index`, which is needed to perform some kinds of operations in the api, like getting transactions information.


### Fullnode Wallet 2
This fullnode will be used by another of our `hathor-wallet-headless` instances, and we are calling it `fullnode-wallet-2`.

```sh
mkdir -p ~/hathor-private-tutorial/fullnode-wallet-2

docker run -ti --network="host" -v ~/hathor-private-tutorial/fullnode-wallet-2:/data:consistent -v ~/hathor-private-tutorial/conf:/privnet/conf --env PYTHON_PATH=/privnet/conf --env HATHOR_CONFIG_FILE=privnet.conf.privnet hathornetwork/hathor-core run_node --cache --status 8082 --data /data --x-fast-init-beta --wallet-index --bootstrap tcp://127.0.0.1:40403
```

This command configures the fullnode to:
- Expose its HTTP api on port 8082
- Connect to `tcp://127.0.0.1:40403` (fullnode-main) to sync with the P2P network
- Use `~/hathor-private-tutorial/fullnode-wallet-2` as its data directory in the host.
- Use the configuration file we create for the network in `hathor.conf.privnet`
- Enable `wallet index`, which is needed to perform some kinds of operations in the api, like getting transactions information.

### Fullnode Miner
This fullnode will be used by our miner, and we will call it `fullnode-miner` for that reason.

```sh
mkdir -p ~/hathor-private-tutorial/fullnode-miner

docker run -ti --network="host" -v ~/hathor-private-tutorial/fullnode-miner:/data:consistent -v ~/hathor-private-tutorial/conf:/privnet/conf --env PYTHON_PATH=/privnet/conf --env HATHOR_CONFIG_FILE=privnet.conf.privnet hathornetwork/hathor-core run_node --cache --status 8083 --stratum 8093 --data /data --x-fast-init-beta --bootstrap tcp://127.0.0.1:40403
```

This command configures the fullnode to:
- Expose its HTTP api on port 8083
- Expose its miing api on port 8093
- Connect to `tcp://127.0.0.1:40403` (fullnode-main) to sync with the P2P network
- Use `~/hathor-private-tutorial/fullnode-miner` as its data directory in the host.
- Use the configuration file we create for the network in `hathor.conf.privnet`

# Running the miner
Now that our fullnodes are in place, the next step is to have a miner to mine blocks in our new private network.

The easiest way to do this in a local testing environment is to run a CPU Miner.
We have a CPU Miner as a Docker image in https://hub.docker.com/r/hathornetwork/cpuminer

You should run it with the following command, replacing `<address>` by the same address we generated in the beginning of the guide:
```sh
docker run -ti --network="host" hathornetwork/cpuminer -t 1 -a sha256d -o stratum+tcp://127.0.0.1:8093 --coinbase-addr <address>
```

This command is telling the miner to connect to our `fullnode-miner` in the port 8093.

When you start mining, you'll see that all fullnodes start logging the new blocks being received. The logs of the `fullnode-miner` will be different, because the miner will be connected directly to it sending mined blocks. The other fullnodes will only receive the new blocks through the sync algorithm.

Also note that all fullnodes will receive the blocks, even though only the `fullnode-main` is directly connected to the `fullnode-miner`.

# Running a TxMiningService
The role of this service is to act as a miner for transactions generated by wallets. In Hathor, transactions are also mined just like blocks, just with a lower difficulty.

Run it like the following. You should replace the `<address>` with the same you generated in the beginning for the genesis block:
```sh
docker run -it --network="host" hathornetwork/tx-mining-service http://127.0.0.1:8083 --address <address> --api-port 8100 --stratum-port 8101 --testnet
```

Run a CPU miner for the TxMiningService, using the same address:
```sh
docker run -ti --network="host" hathornetwork/cpuminer -t 1 -a sha256d -o stratum+tcp://127.0.0.1:8101 --coinbase-addr <address>
```

When running the wallets, we will configure them to use this service for transaction mining.

# Running our wallets
<!--
We will run 2 wallet-headles to demonstrate the wallet functionality. They are basically wallets that run as a HTTP api and have most of the same functions as the desktop or mobile wallets.

Each wallet-headless will be connected to one of our fullnodes dedicated to wallets (`fullnode-wallet-1` and `fullnode-wallet-2`). But more than one wallet can use the same fullnode without problem. -->

We will run one wallet-headless to demonstrate the wallet functionality. It's basically a wallet that runs as a HTTP api and have most of the same functions as the desktop or mobile wallets.

The first step is to clone the repo https://github.com/HathorNetwork/hathor-wallet-headless

## Running the wallet


## Running the wallet
We will run the wallet using the seed we generated at the beginning of the guide. This seed should have some balance, since we configured it as the destination for mining rewards.

Run it with this, replacing `<seed>` with the seed from the beginning of the guide:

```
cat << EOF > .env1
HEADLESS_HTTP_PORT=8000
HEADLESS_NETWORK=testnet
HEADLESS_SERVER=http://127.0.0.1:8081/v1a/
HEADLESS_SEED_DEFAULT=<see>
HEADLESS_TX_MINING_URL=http://127.0.0.1:8100/
EOF

docker run -it --network="host" --env-file=.env1 hathornetwork/hathor-wallet-headless
```

Start the wallet:
```sh
curl -X POST --data "wallet-id=wallet1" --data "seedKey=default" http://localhost:8000/start
```

Check the wallet balance:
```sh
curl -H "X-Wallet-Id: wallet1" http://localhost:8000/wallet/balance
```

Get address to send transaction:
```sh
curl -H "X-Wallet-Id: wallet1" http://localhost:8000/wallet/address
```

Send a transaction, replacing the address from the last command output:

```sh
curl -X POST -H "X-Wallet-Id: wallet1" --data "address=<address>" --data "value=101" http://localhost:8000/wallet/simple-send-tx
```

Congratulations, you have successfully sent a transaction in the network, and now has a fully functional private network!


<!-- ## Second wallet
Run it with:

```
cat << EOF > .env2
HEADLESS_HTTP_PORT=8001
HEADLESS_NETWORK=testnet
HEADLESS_SERVER=http://127.0.0.1:8082/v1a/
HEADLESS_SEED_DEFAULT=<seed>
EOF

docker run -it --network="host" --env-file=.env2 hathornetwork/hathor-wallet-headless
```

Create the seed for the wallet. You will use it in the next command.
```
docker run --entrypoint npm hathornetwork/hathor-wallet-headless run generate_words
```

Start the wallet:
```
curl -X POST --data "wallet-id=wallet1" --data "seed=<seed>" http://localhost:8001/start
``` -->


# Running the Explorer
Clone the repo: https://github.com/HathorNetwork/hathor-explorer

Install dependencies:
```
npm install
```

Run the frontend:
```
export REACT_APP_BASE_URL=http://localhost:8080/v1a/
export REACT_APP_WS_URL=ws://localhost:8080/v1a/ws/

npm start
```

Please note that the "Network" page of the Explorer will not work in this tutorial, because it needs additional backend services that we decided to not include here. The other features should work normally.

# Customizing the network

There are a lot more parameters that can be customized in the network if needed.

The most up-to-date source to learn which parameters are available is the source code itself, which can be checked on https://github.com/HathorNetwork/hathor-core/blob/master/hathor/conf/settings.py

You can add any of those parameters in the file `~/hathor-private-tutorial/conf/privnet.py` that we created ealier, then restart the fullnodes for the changes to take effect.

Beware you should be careful when changing any of those settings in fullnodes connected to Hathor mainnet/testnet, since changing many of them can result in forks in the network or other problems. For private networks it's safe to change them.

Some of the most important are:
- `BOOTSTRAP_DNS`: Used to make the fullnode discover other peers to connect to using DNS. The DNS query can return a list of other fullnodes connection strings.
- `ENABLE_PEER_WHITELIST`: You can use this if you need to make sure only connections from authorized fullnodes are accepted. The control is done through the `peer-id` of each fullnode.
- `DECIMAL_PLACES`: How many decimal places the network will allow in transaction values.
- `BLOCKS_PER_HALVING`: How many blocks until a reward halving occurs.
- `AVG_TIME_BETWEEN_BLOCKS` : The time between blocks
- `REWARD_SPEND_MIN_BLOCKS`: The number of blocks before the mining reward can be spent.

A lot of other parameters can be checked in the mentioned file.