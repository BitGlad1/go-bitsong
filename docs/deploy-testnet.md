# Deploy Your Own BitSong Testnet

This document describes 3 ways to setup a network of `bitsongd` nodes, each serving a different usecase:

1. Single-node, local, manual testnet
2. Multi-node, local, automated testnet
3. Multi-node, remote, automated testnet

Supporting code can be found in the [networks directory](https://github.com/BitSongOfficial/go-bitsong/tree/master/networks) and additionally the `local` or `remote` sub-directories.

> NOTE: The `remote` network bootstrapping may be out of sync with the latest releases and is not to be relied upon.

## Available Docker images

> NOTE: this part is work in progress....

In case you need to use or deploy go-bitsong as a container you could skip the `build` steps and use the official images, $TAG stands for the version you are interested in:

- `docker run -it -v ~/.bitsongd:/root/.bitsongd -v ~/.bitsongcli:/root/.bitsongcli tendermint:$TAG bitsongd init`
- `docker run -it -p 26657:26657 -p 26656:26656 -v ~/.bitsongd:/root/.bitsongd -v ~/.bitsongcli:/root/.bitsongcli tendermint:$TAG bitsongd start`
- ...
- `docker run -it -v ~/.bitsongd:/root/.bitsongd -v ~/.bitsongcli:/root/.bitsongcli tendermint:$TAG bitsongcli version`

The same images can be used to build your own docker-compose stack.

## Single-node, Local, Manual Testnet

This guide helps you create a single validator node that runs a network locally for testing and other development related uses.

### Requirements

- [Install go-bitsong](./installation.md)
- [Install `jq`](https://stedolan.github.io/jq/download/) (optional)

### Create Genesis File and Start the Network

```bash
# You can run all of these commands from your home directory
cd $HOME

# Initialize the genesis.json file that will help you to bootstrap the network
bitsongd init --chain-id=testing testing

# Optionally set the `bitsongcli` config
bitsongcli config chain-id bitsong-testnet-1
bitsongcli config output json
bitsongcli config indent true
bitsongcli config trust-node true

# Create a key to hold your validator account
bitsongcli keys add validator

# Change default bond token genesis.json
sed -i 's/stake/ubtsg/g' ~/.bitsongd/config/genesis.json

# Add that key into the genesis.app_state.accounts array in the genesis file
# NOTE: this command lets you set the number of coins. Make sure this account has some coins
# with the genesis.app_state.staking.params.bond_denom denom, the default is staking
bitsongd add-genesis-account $(bitsongcli keys show validator -a) 1000000000000ubtsg

# Generate the transaction that creates your validator
bitsongd gentx --name validator

# Add the generated bonding transaction to the genesis file
bitsongd collect-gentxs

# Now its safe to start `bitsongd`
bitsongd start
```

This setup puts all the data for `bitsongd` in `~/.bitsongd`. You can examine the genesis file you created at `~/.bitsongd/config/genesis.json`. With this configuration `bitsongcli` is also ready to use and has an account with tokens (both staking and custom).

## Multi-node, Local, Automated Testnet

> NOTE: this part is work in progress....

From the [networks/local directory](https://github.com/BitSongOfficial/go-bitsong/tree/master/networks/local):

### Requirements

- [Install go-bitsong](./installation.md)
- [Install docker](https://docs.docker.com/engine/installation/)
- [Install docker-compose](https://docs.docker.com/compose/install/)

### Build

> NOTE: this part is work in progress....

Build the `bitsongd` binary (linux) and the `tendermint/bitsongdnode` docker image required for running the `localnet` commands. This binary will be mounted into the container and can be updated rebuilding the image, so you only need to build the image once.

```bash
# Work from the SDK repo
cd $GOPATH/src/github.com/BitSongOfficial/go-bitsong

# Build the linux binary in ./build
make build-linux

# Build tendermint/bitsongdnode image
make build-docker-bitsongdnode
```

### Run Your Testnet

> NOTE: this part is work in progress....

To start a 4 node testnet run:

```
make localnet-start
```

This command creates a 4-node network using the bitsongdnode image.
The ports for each node are found in this table:

| Node ID | P2P Port | RPC Port |
| --------|-------|------|
| `bitsongnode0` | `26656` | `26657` |
| `bitsongnode1` | `26659` | `26660` |
| `bitsongnode2` | `26661` | `26662` |
| `bitsongnode3` | `26663` | `26664` |

To update the binary, just rebuild it and restart the nodes:

```
make build-linux localnet-start
```

### Configuration

The `make localnet-start` creates files for a 4-node testnet in `./build` by
calling the `bitsongd testnet` command. This outputs a handful of files in the
`./build` directory:

```bash
$ tree -L 2 build/
build/
├── bitsongcli
├── bitsongd
├── gentxs
│   ├── node0.json
│   ├── node1.json
│   ├── node2.json
│   └── node3.json
├── node0
│   ├── bitsongcli
│   │   ├── key_seed.json
│   │   └── keys
│   └── bitsongd
│       ├── ${LOG:-bitsongd.log}
│       ├── config
│       └── data
├── node1
│   ├── bitsongcli
│   │   └── key_seed.json
│   └── bitsongd
│       ├── ${LOG:-bitsongd.log}
│       ├── config
│       └── data
├── node2
│   ├── bitsongcli
│   │   └── key_seed.json
│   └── bitsongd
│       ├── ${LOG:-bitsongd.log}
│       ├── config
│       └── data
└── node3
    ├── bitsongcli
    │   └── key_seed.json
    └── bitsongd
        ├── ${LOG:-bitsongd.log}
        ├── config
        └── data
```

Each `./build/nodeN` directory is mounted to the `/bitsongd` directory in each container.

### Logging

Logs are saved under each `./build/nodeN/bitsongd/go-bitsong.log`. You can also watch logs
directly via Docker, for example:

```
docker logs -f bitsongdnode0
```

### Keys & Accounts

To interact with `bitsongcli` and start querying state or creating txs, you use the
`bitsongcli` directory of any given node as your `home`, for example:

```shell
bitsongcli keys list --home ./build/node0/bitsongcli
```

Now that accounts exists, you may create new accounts and send those accounts
funds!

::: tip
**Note**: Each node's seed is located at `./build/nodeN/bitsongcli/key_seed.json` and can be restored to the CLI using the `bitsongcli keys add --restore` command
:::

### Special Binaries

If you have multiple binaries with different names, you can specify which one to run with the BINARY environment variable. The path of the binary is relative to the attached volume. For example:

```
# Run with custom binary
BINARY=bitsongfoo make localnet-start
```

## Multi-Node, Remote, Automated Testnet

> NOTE: this part is work in progress....

The following should be run from the [networks directory](https://github.com/BitSongOfficial/go-bitsong/tree/master/networks).

### Terraform & Ansible

Automated deployments are done using [Terraform](https://www.terraform.io/) to create servers on AWS then
[Ansible](http://www.ansible.com/) to create and manage testnets on those servers.

### Prerequisites

- Install [Terraform](https://www.terraform.io/downloads.html) and [Ansible](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) on a Linux machine.
- Create an [AWS API token](https://docs.aws.amazon.com/general/latest/gr/managing-aws-access-keys.html) with EC2 create capability.
- Create SSH keys

```
export AWS_ACCESS_KEY_ID="2345234jk2lh4234"
export AWS_SECRET_ACCESS_KEY="234jhkg234h52kh4g5khg34"
export TESTNET_NAME="remotenet"
export CLUSTER_NAME= "remotenetvalidators"
export SSH_PRIVATE_FILE="$HOME/.ssh/id_rsa"
export SSH_PUBLIC_FILE="$HOME/.ssh/id_rsa.pub"
```

These will be used by both `terraform` and `ansible`.

### Create a Remote Network

```
SERVERS=1 REGION_LIMIT=1 make validators-start
```

The testnet name is what's going to be used in --chain-id, while the cluster name is the administrative tag in AWS for the servers. The code will create SERVERS amount of servers in each availability zone up to the number of REGION_LIMITs, starting at us-east-2. (us-east-1 is excluded.) The below BaSH script does the same, but sometimes it's more comfortable for input.

```
./new-testnet.sh "$TESTNET_NAME" "$CLUSTER_NAME" 1 1
```

### Quickly see the /status Endpoint

```
make validators-status
```

### Delete Servers

```
make validators-stop
```

### Logging

You can ship logs to Logz.io, an Elastic stack (Elastic search, Logstash and Kibana) service provider. You can set up your nodes to log there automatically. Create an account and get your API key from the notes on [this page](https://app.logz.io/#/dashboard/data-sources/Filebeat), then:

```
yum install systemd-devel || echo "This will only work on RHEL-based systems."
apt-get install libsystemd-dev || echo "This will only work on Debian-based systems."

go get github.com/mheese/journalbeat
ansible-playbook -i inventory/digital_ocean.py -l remotenet logzio.yml -e LOGZIO_TOKEN=ABCDEFGHIJKLMNOPQRSTUVWXYZ012345
```

### Monitoring

You can install the DataDog agent with:

```
make datadog-install
```
