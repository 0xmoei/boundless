# Boundless Prover Guide
The Boundless Prover Node is a computational proving system that participates in the Boundless decentralized proving market. Provers stake USDC, bid on computational tasks, generate zero-knowledge proofs using GPU acceleration, and earn rewards for successful proof generation.

This guide covers both **automated** and **manual** installation methods for Ubuntu 20.04/22.04 systems.

## Table of Contents
- [Boundless Prover Market](#boundless-prover-market)
- [Notes](#notes)
- [Requirements](#requirements)
- [Rent GPU](#rent-gpu)
- [Automated Setup](#automated-setup)
- [Manual Setup](#manual-setup)
  - [Dependencies](#dependencies)
  - [System Hardware Check](#system-hardware-check)
  - [Configure Prover](#configure-prover)
  - [Running Prover](#running-prover)
  - [Run Bento](#run-bento)
  - [Configure Network](#configure-network)
  - [Deposit Stake](#deposit-stake)
  - [Run Broker](#run-broker)
- [Bento (Prover) & Broker Optimizations](#bento-prover--broker-optimizations)
  - [Segment Size (Prover)](#segment-size-prover)
  - [Benchmarking Bento](#benchmarking-bento)
  - [Broker Optimization](#broker-optimization)
  - [Multi Brokers](#multi-brokers)
- [Safe Update or Stop Prover](#safe-update-or-stop-prover)
- [Debugging](#debugging)

---

## Boundless Prover market
First, you need to know how **Boundless Prover market** actually works to realize what you are doing.

1. **Request Submission**: Developers submit computational tasks as "orders" on Boundless, offering ETH/ERC-20 rewards
2. **Prover Stakes USDC**: Provers must deposit `USDC` as stake before bidding on orders
3. **Bidding Process**: Provers detect orders and submit competitive bids (`mcycle_price`)
4. **Order Locking**: Winning provers lock orders using staked USDC, committing to prove within deadline
5. **Proof Generation**: Provers compute and submit proofs using GPU acceleration
6. **Rewards/Slashing**: Valid proofs earn rewards; invalid/late proofs result in stake slashing

---

## Notes
- The prover is in beta phase, while I admit that my guide is really perfect, you may get some troubles in the process of running it, so you can wait until official incentivized testnet with more stable network and more updates to this guide, or start exprimenting now.
- I advice to start with testnet networks due to loss of stake funds
- I will update this github guide constantly, so you always have to check back here later and follow me on [X](https://x.com/0xMoei) for new updates.

---

## Requirements
### Hardware
* CPU - 16 threads, reasonable single core boost performance (>3Ghz)
* Memory - 32 GB
* Disk - 100 GB NVME/SSD
* GPU
  * Minimum: one 8GB vRAM GPU
  * Recommended to be competetive: 10x GPU with min 8GB vRAM
  * Recomended GPU models are 4090, 5090 and L4.
> * You better test it out with single GPUs by lowering your configurations later by reading the further sections.

### Software
* Supported: Ubuntu 20.04/22.04
* No support: Ubuntu 24.04
* If you are running on Windows os locally, install Ubuntu 22 WSL using this [Guide](https://github.com/0xmoei/Install-Linux-on-Windows)

---

## Rent GPU

**Recommended GPU Providers**
* **[Vast.ai](https://cloud.vast.ai/?ref_id=62897&creator_id=62897&name=Ubuntu%2022.04%20VM)**: SSH-Key needed
  * Rent **VM Ubuntu** [template](https://cloud.vast.ai/?ref_id=62897&creator_id=62897&name=Ubuntu%2022.04%20VM)
  * Refer to this [Guide](https://github.com/0xmoei/Rent-and-Config-GPU) to generate SSH-Key, Rent GPU and connect to your Vast GPU

---

# Automated Setup
For an automated installation and prover management, you can use this script that handles all dependencies, configuration, setup, and prover management automatically.

## Download and Run the Installer:
```bash
# Update packages
apt update && apt upgrade -y

# download wget
apt install wget
```

```bash
# Download the installation script
wget https://raw.githubusercontent.com/0xmoei/boundless/main/install_prover.sh -O install_prover.sh

# Make it executable
chmod +x install_prover.sh

# Run the installer
./install_prover.sh
```
* Installation may take time since we are installing drivers and building big files, so no worries.

### During Installation:
* The script will automatically detect your GPU configuration
* You'll be prompted for:
  * Network selection (mainnet/testnet)
  * RPC URL: Read [Get RPC](#get-rpc) for more details
  * Private key (input is hidden)
  * Broker config parameters: Visit [Broker Optimization](#broker-optimization) to read parameters details


### Post-Installation Management Script:
After installation, to Run or Configure your Prover, you have to navigate to the installation directory and run Management Script `prover.sh`:

```bash
cd ~/boundless
./prover.sh
```
The management script provides a menu with:
- **Service Management**: Start/stop broker, view logs, health checks
- **Configuration**: [Change network](#get-rpc), update private key, edit [broker config](#broker-optimization)
- **Stake Management**: Deposit USDC stake, check balance
- **Performance Testing**: Run benchmarks with order IDs
- **Monitoring**: Real-time GPU monitoring


### Modify CPU/RAM of x-exec-agent-common & gpu-prove-agent
The `prover.sh` script manages all broker configurations (.e.g `broker.toml`), but to optimize and add some RAM and CPU to your `compose.yml`, you can navigate to [x-exec-agent-common](#modify-cpu/ram-of-gpu_prove_agent) & [gpu-prove-agent sections](modify-cpu/ram-of-gpu_prove_agent)
* Re-run your broker after doing configurations to `compose.yml`

### Note
Even if you setup using automated script, I recommend you to read **[Manual Setup](#manual-setup)** and **[Bento (Prover) & Broker Optimizations](#bento-prover--broker-optimizations)** sections to learn to optimize your prover.

---

# Manual Setup
Here is the step by step guide to Install and run your Prover smoothly, but please pay attention to these notes:
* Read every single word of this guide, if you really want to know what you are doing.
* There is an [Prover+Broker Optimization](https://github.com/0xmoei/boundless/blob/main/README.md#bento-prover--broker-optimizations) section where you need to read after setting up prover.


## Dependecies
### Delete Preserved Variables

* Open `/etc/environment`:
```bash
sudo nano /etc/environment
```
Delete everything.

* Add this code to it:
```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

### Install & Update Packages
```bash
apt update && apt upgrade -y
apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev tar clang bsdmainutils ncdu unzip libleveldb-dev libclang-dev ninja-build -y
```

### Clone Boundless Repo
```bash
git clone https://github.com/boundless-xyz/boundless
cd boundless
git checkout release-0.11
```

### Install Dependecies
To run a Boundless prover, you'll need the following dependencies:
* Docker compose
* GPU Drivers
* Docker Nvidia Support
* Rust programming language
* `Just` command runner
* CUDA Tollkit

For a quick set up of Boundless dependencies on Ubuntu 22.04 LTS, you can run:
```bash
bash ./scripts/setup.sh
```
However, we need to install some dependencies manually:

```console
\\ Execute command lines one by one

# Install rustup:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
. "$HOME/.cargo/env"

# Update rustup:
rustup update

# Install the Rust Toolchain:
apt update
apt install cargo

# Verify Cargo:
cargo --version

# Install rzup:
curl -L https://risczero.com/install | bash
source ~/.bashrc

# Verify rzup:
rzup --version

# Install RISC Zero Rust Toolchain:
rzup install rust

# Install cargo-risczero:
cargo install cargo-risczero
rzup install cargo-risczero

# Update rustup:
rustup update

# Install Bento-client:
cargo install --locked --git https://github.com/risc0/risc0 bento-client --branch release-2.1 --bin bento_cli
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify Bento-client:
bento_cli --version

# Install Boundless CLI:
cargo install --locked boundless-cli
export PATH=$PATH:/root/.cargo/bin
source ~/.bashrc

# Verify boundless-cli:
boundless -h

# Install Just
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
cargo install just
just --version
```

---

## System Hardware Check
* In the beginning, to configure your Prover, You need to know what's your GPUs IDs (if multiple GPUs), CPU cores and RAM.
* Also the following tools are best to monitor your hardware during proving.

### GPU Check:
* If your Nvidia driver and Cuda tools are installed succesfully, run the following command to see your GPUs status:
```
nvidia-smi
```
* You can now monitor Nvidia driver & Cuda Version, GPU utilization & memory usage.
* In the image below, there are four GPUs with **0-3** IDs, you'll need it when adding GPU to your configuration.

![image](https://github.com/user-attachments/assets/26c57f43-0fbf-4068-949c-b2ea31273998)

* Check your system GPUs IDs (e.g. 0 through X):
```bash
nvidia-smi -L
```

### Number of CPU Cores and Threads:
```
lscpu
```

### CPU & RAM check (Realtime):
To see the status of your CPU and RAM.
```bash
htop
```

### GPU Check (Realtime):
The best for real-time monitoring your GPUs in a seprated terminal while your prover is proving.
```bash
nvtop
```

---

## Configure Prover
### Single GPU:
The default `compose.yml` file defines all services within Prover.
* Default `compose.yml` only supporting single-GPU and default CPU, RAM utilization.
* Edit `compose.yml` by this command:
  ```bash
  nano compose.yml
  ```
* The current `compose.yml` is set for `1` GPU by default, you can bypass editing it if you only have one GPU.
* In single GPUs, you can increase the RAM & CPU of the `x-exec-agent-common` and `gpu_prove_agent0` services in `compose.yml` instead to maximize the utilization of your system

### Multiple GPUs
* 4 GPUs:
To add more GPUs or modify CPU and RAM sepcified to each GPU, replace the current compose file with my [custom compose.yml](https://github.com/0xmoei/boundless/blob/main/compose.yml) with 4 custom GPUs

* More/Less than 4 GPUs:
Follow this [detailes step by step guide](https://github.com/0xmoei/boundless/blob/main/add-remove-gpu.md) to add or remove the number of 4 GPUs in my custom `compose.yml` file

---

## Configure Segment Size
Larger segment size caused more proving (bento) performance, but require more GPU VRAM. To pick the right `SEGMENT_SIZE` for your GPU VRAM, see the [official performance optimization page](https://docs.beboundless.xyz/provers/performance-optimization#finding-the-maximum-segment_size-for-gpu-vram).

![image](https://github.com/user-attachments/assets/ef566e27-ce69-4563-a035-87733827126d)

* Note, when you set a number for `SEGMENT_SIZE`, it sets that number for each GPU identically.
* The default value of `SEGMENT_SIZE` is `21`, if you have a 24GB vRAM GPU, skip this step

### Setting SEGMENT_SIZE
**Configure `SEGMENT_SIZE` in `compose.yml`**

* `SEGMENT_SIZE` in `compose.yml` under the `x-exec-agent-common` service is `21`by default.
  * You can change the value of `SEGMENT_SIZE` directly in `compose.yml` by adding `SEGMENT_SIZE: 21` in `environment` section of `x-exec-agent-common`
  * Your `x-exec-agent-common` will be like this:
  ```yaml
  x-exec-agent-common: &exec-agent-common
  <<: *agent-common
  mem_limit: 4G
  cpus: 3
  environment:
    <<: *base-environment
    RISC0_KECCAK_PO2: ${RISC0_KECCAK_PO2:-17}
    SEGMENT_SIZE: 19
  entrypoint: /app/agent -t exec --segment-po2 ${SEGMENT_SIZE:-21}
  ```
  > `entrypoint` uses `${SEGMENT_SIZE:-21}`, a shell parameter expansion that sets the segment size to 21 unless `SEGMENT_SIZE` is defined in the container’s `environment`.

**Two other options to Configure `SEGMENT_SIZE`**
* You can add `SEGMENT_SIZE` variable with its value to the preserved network `.env` files like `.env.base-sepolia`,`.env.broker`, etc. if you are using them.
* While it's not recommended, you simply can replace `${SEGMENT_SIZE:-21}` in `compose.yml` with the number itself like `entrypoint: /app/agent -t exec --segment-po2 21`

---

## Running Prover
Boundless is comprised of two major components:
* `Bento` is the local proving infrastructure. Bento will take the locked orders from `Broker`, prove them and return the result to `Broker`.
* `Broker` interacts with the Boundless market. `Broker` can submit or request proves from the market.

---

## Run Bento
To get started with a test proof on a new proving machine, let's run `Bento` to benchmark our GPUs:
```bash
just bento
```
* This will spin up `bento` without the `broker`.

Check the logs :
```bash
just bento logs
```

Run a test proof:
```bash
RUST_LOG=info bento_cli -c 32
```
* If everything works well, you should see something like the following as `Job Done!`:

![image](https://github.com/user-attachments/assets/a67fdfb0-3d22-4a4a-b47a-247567df0d45)

* To check if all your GPUs are utilizing:
  *  Increase `32` to `1024`/`2048`/`4096`
  *  Open new terminal with `nvtop` command
  *  Run the test proof and monitor your GPUs utilization.

---

## Get RPC
* According to what network you want to run your prover on, you'll need an RPC endpoint that supports `eth_newBlockFilter` event.
  * You can search for `eth_newBlockFilter` in the documents of third-party RPC providers to see if they support it or not.

RPC providers I know they support `eth_newBlockFilter` and I recommend:
* [Alchemy](https://dashboard.alchemy.com/apps):
  * Alchemy is the best provider so far
* [BlockPi](https://dashboard.blockpi.io/):
  * Support free Base Mainnet, Base Sepolia. ETH sepolia costly as $49
* [Chainstack](https://console.chainstack.com/):
  * You have to change the value of `lookback_blocks` from `300` to `0`, because chainstack's free plan doesn't support `eth_getlogs`, so you won't be able to check last 300 blocks for open orders at startup (which is not very important i believe)
  * Check **Broker Optimization** section to know how to change `lookback_blocks` value in `broker.toml`
* Run your own RPC node:
  * This is actually the best way but costly in terms of needing ~550-650 GB Disk
  * [Guide for ETH Sepolia](https://github.com/0xmoei/geth-prysm-node/blob/main/README.md)
* Quicknode supports `eth_newBlockFilter` but was NOT compatible with prover somehow idk. It blew up my prover.

---

## Set Network and Wallet
Boundless is currently available on `Base Mainnet`, `Base Sepolia` and `Ethereum Sepolia`.
### Method 1: Environment Variables
Before running prover, simply execute these commands:
```bash
export RPC_URL="your-rpc-url"
export PRIVATE_KEY=your-private-key
```
* Replace `your-rpc-url` & `your-private-key` without `0x` perfix, and execute the commands

### Method 2: .env files
I **recommend** to go through **Method 1** and skip this step to [Deposit Stake](#deposit-stake), otherwise you can follow method 2 by going [here](https://github.com/0xmoei/boundless/tree/main?tab=readme-ov-file#network-configurations-method-2-env-files)

---

## Deposit Stake
Provers will need to deposit` USDC` to the Boundless Market contract to use as stake when locking orders.

Note that `USDC` has a different address on each network. Refer to the [Deployments page](https://docs.beboundless.xyz/developers/smart-contracts/deployments) for the addresses. USDC can be obtained on testnets from the [Circle Faucet](https://faucet.circle.com/). You can alsi [Bridge](https://superbridge.app/base-sepolia) USDC.

**Add `boundless` CLI to bash:**
```
source ~/.bashrc
```

**Deposit Stake:**
```
boundless account deposit-stake STAKE_AMOUNT
```
* Deposit Stake Balance: `boundless account stake-balance`

---

##  Run Broker
You can now start `broker` (which runs both `bento` + `broker` i.e. the full proving stack!):
```bash
just broker
```
Check the total proving logs:
```bash
just broker logs
```
Check the `broker` logs, which has the most important logs of your `order` lockings and fulfillments:
```
docker compose logs -f broker

# For last 100 logs
docker compose logs -fn 100
```

![image](https://github.com/user-attachments/assets/c7e8e343-ec4c-4202-b4ba-ef1cf04cedaa)

* You may stuck at `Subscribed to offchain Order stream`, but it starts detecting orders soon.

---

# Bento (Prover) & Broker Optimizations
There are many factors to be optimized to win in provers competetion where you can read the official guide for [broker](https://docs.beboundless.xyz/provers/broker) or [prover](https://docs.beboundless.xyz/provers/performance-optimization)

## Boost pre-flight execution
* `exec_agent` services in `compose.yml` is doing the preflight execution of orders to estimate if prover can bid on them or not.
* They are important in preflighting orders concurrently and locking them faster to compete with other provers.
  * More `exec_agent` will preflight more orders concurrently.
  * More CPU/RAM in a single `exec_agent` will preflight orders faster.
* Increasing it from default value: `2` depends on how many concurrent preflight execution you want to allow.
* We have two services related to exec agents: `x-exec-agent-common` and `exec_agent`
  * `x-exec-agent-common`: covers the main settings of all `exec_agent` services including CPU and Memory sepecified to each
  * `exec_agentX`: They are the agents themselves that you can addup for more concurrent preflight execution. `X` is the number of the agents you want to specify. To add more, you increase `X` by `+1`

Example of `x-exec-agent-common` in your `compose.yml`:
```yml
x-exec-agent-common: &exec-agent-common
  <<: *agent-common
  mem_limit: 4G
  cpus: 2
  environment:
    <<: *base-environment
    RISC0_KECCAK_PO2: ${RISC0_KECCAK_PO2:-17}
  entrypoint: /app/agent -t exec --segment-po2 ${SEGMENT_SIZE:-21}
```
* You can increase `cpus` and `mem_limit`

Example of `exec_agent` in your `compose.yml`:
```yaml
  exec_agent0:
    <<: *exec-agent-common

  exec_agent1:
    <<: *exec-agent-common
```
* To increase agents, addup more lines of these and increase their number by `+1`

## Boost Proving GPUs
* `gpu_prove_agent` service in your `compose.yml` handles proving the orders after they got locked by utilizing your GPUs.
* In single GPUs, you can increase performance by increasing CPU/RAM of GPU agents.
* The default number of its CPU and RAM is fine but if you have good system spec, you can increase them for each GPU.
* You see smth like below code as your `gpu_prove_agentX` service in your `compose.yml` where you can increase the memory and cpu cores of each gpu agent.
   ```yml
     gpu_prove_agent0:
       <<: *agent-common
       runtime: nvidia
       mem_limit: 4G
       cpus: 4
       entrypoint: /app/agent -t prove
       deploy:
         resources:
           reservations:
             devices:
               - driver: nvidia
                 device_ids: ['0']
                 capabilities: [gpu]
   ```
* While the default CPU/RAM for each GPU is enough, for single GPUs, you can increase them to increase efiiciency, but don't maximize and always keep some CPU/RAML for other jobs.

## Benchmarking Bento
Install psql:
```bash
apt update
apt install postgresql-client
psql --version
```

**1. Recommended: Benchmark by simulating an order id: (make sure Bento is running):**
```bash
boundless proving benchmark --request-ids <IDS>
```
* You can use the order IDs listed [here](https://explorer.beboundless.xyz/orders)
* You can add multiples by adding comma-seprated ones.
* Recommended to pick a few requests of varying sizes and programs, biased towards larger proofs for a more representative benchmark.

![image](https://github.com/user-attachments/assets/04ca61f7-a658-4cb8-b09b-928bbe4694d4)

* As in the image above, the prover is estimated to handle ~430,000 cycles per second (~430 khz). 
* Use a lower amount of the recommented `peak_prove_khz` in your `broker.toml` (I explain it more in the next step)

> You can use `nvtop` command in a seprated terminal to check your GPU utilizations.

**2. Benchmark using Harness Test**
* Optionally you can benchmark GPUs by a ITERATION_COUNT:.
```
RUST_LOG=info bento_cli -c <ITERATION_COUNT>
```
`<ITERATION_COUNT>` is the number of times the synthetic guest is executed. A value of `4096` is a good starting point, however on smaller or less performant hosts, you may want to reduce this to `2048` or `1024` while performing some of your experiments. For functional testing, `32` is sufficient.

* Check `khz` &  `cycles` proved in the harness test
```
bash scripts/job_status.sh JOB_ID
```
* replace `JOB_ID` with the one prompted to you when running a test.
* Now you get the `hz` which has to be devided by 1000x to be in `khz` and the `cycles` it proved.
* If got error `not_found`, it's cuz you didn't create `.env.broker` and the script is using the `SEGMENT_SIZE` value in `.env.broker` to query your Segment size, do `cp .env.broker-template .env.broker` to fix.

---

## Broker Optimization

* Broker is one of the containers of the prover, it's not proving itself, it's for onchain activities, and initializing with orders like locking orders or setting amount of stake bids, etc.
* `broker.toml` has the settings to configure how your broker interact on-chain and compete with other provers.

Copy the template to the main config file:
```bash
cp broker-template.toml broker.toml
```

Edit broker.toml file:
```bash
nano broker.toml
```
* You can see an example of the official `broker.toml` [here](https://github.com/boundless-xyz/boundless/blob/main/broker-template.toml)

### Increasing Lock-in Rate
Once your broker is running, before the gpu-based prover gets into work, broker must compete with other provers to lock-in the orders. Here is how to optimize broker to lock-in faster than other provers:

1. Decreasing the `mcycle_price` would tune your Broker to `bid` at lower prices for proofs.
* Once an order detected, the broker runs a preflight execution to estimate how many `cycles` the request needs. As you see in the image, a prover proved orders with millions or thousands of cycles.
* `mcycle_price` is actually price of a prover for proving each 1 million cycles. Final price = `mcycle_price` x `cycles`
* The less you set `mcycle_price`, the higher chance you outpace other provers.

![image](https://github.com/user-attachments/assets/fab9cc79-362f-4a43-a461-258ffe0bfc1a)


* To get idea of what `mcycle_price` are other provers using, find an order in [explorer](https://explorer.beboundless.xyz/orders/0xc2db89b2bd434ceac6c74fbc0b2ad3a280e66db024d22ad3) with your prefered network, go to details page of the order and look for `ETH per Megacycle`

![image](https://github.com/user-attachments/assets/6dd0c012-bff7-4a98-97ae-3cdfb288bc43)


2. Increasing `lockin_priority_gas` to consume more gas to outrun other bidders. You might need to first remove `#` to uncomment it's line, then set the gas. It's based on Gwei.

### Other settings in `broker.toml`
Read more about them in [official doc](https://docs.beboundless.xyz/provers/broker#settings-in-brokertoml)
* `peak_prove_khz`: Maximum number of cycles per second (in kHz) your proving backend can operate.
  * You can set the `peak_prove_khz` by following the previous step [(Benchmarking Bento)](https://github.com/0xmoei/boundless/tree/main#benchmarking-bento)

* `max_concurrent_proofs`: Maximum number of orders the can lock. Increasing it, increase the ability to lock more orders, but if you prover cannot prove them in the specified deadline, your stake assets will get slashed.
  * When the numbers of running proving jobs reaches that limit, the system will pause and wait for them to get finished instead of locking more orders.
  * It's set as `2` by default, and really depends on your GPU and your configuration, you have to test it out if you want to inscrease it.

* `min_deadline`: Min seconds left before the deadline of the order to consider bidding on a request or not.
  * Requesters set a deadline for their order, if a prover can't prove during this, it gets slashed.
  * By setting the min deadline, your prover won't accept requests with a deadline less than that.
  * As in the following image of an order in [explorer](https://explorer.beboundless.xyz/), the order fullfilled after the deadline and prover got slashed because of the delay in delivering
 
 ![image](https://github.com/user-attachments/assets/bc497b61-01fe-451a-aeb1-de35efca56af)

---

## Multi Brokers
You can run multiple brokers simultaneously with a single Bento client to generate proofs on different networks.
* You configurations might be different than mine and so you can ask AI chats to modify them. I give you the clues and the example of my current codes
* Generally, you have to make changes in these files: `compose.yml`, `broker.toml`, `.env` files (e.g. `.env.base-sepolia`)

### Modify `compose.yml`

**Step 1:  Add the `broker2` Service**:

In the services section, after your existing `broker` service, add the following `broker2` service. This mirrors the original `broker` configuration but uses a different database and configuration file.
* What do we change in `broker` to add to `broker2`?
 * Name changes to `broker2`
 * `source: ./broker2.toml`
 * `broker2-data:/db/`
 * Updated `--db-url` to `'sqlite:///db/broker2.db'`

**Step 2:  Environment Variables (`.env` files) for Multi-Broker Setup**:

We were using `.env` files (e.g.`.env.base`) for setting the network, we need to link these `.env` files with each broker (.e.g `broker`, `broker1`, `broker3`) in our `compose.yml` file, so each broker runs on the specified network at startup.
* add the following lines after `volumes` of each `broker` service
```
    env_file:
      - .env.base
```

**Step 3:  Add the `broker2-data` Volume**:
* At the end of your `compose.yml`, in the `volumes` section, add the new volume for `broker2`:

For example the `broker`, `broker2` services in my `compose.yml`, supporting two `base` & `eth sepolia` networks with above configurations:

```yaml
  broker:
    restart: always
    depends_on:
      - rest_api
      - gpu_prove_agent0
      - exec_agent0
      - exec_agent1
      - aux_agent
      - snark_agent
      - redis
      - postgres
    profiles: [broker]
    build:
      context: .
      dockerfile: dockerfiles/broker.dockerfile
    mem_limit: 2G
    cpus: 2
    stop_grace_period: 3h
    volumes:
      - type: bind
        source: ./broker.toml
        target: /app/broker.toml
      - broker-data:/db/
    network_mode: host
    env_file:
      - .env.base
    environment:
      RUST_LOG: ${RUST_LOG:-info,broker=debug,boundless_market=debug}
    entrypoint: /app/broker --db-url 'sqlite:///db/broker.db' --set-verifier-address ${SET_VERIFIER_ADDRESS} --boundless-market-address ${BOUNDLESS_MARKET_ADDRESS} --config-file /app/broker.toml --bento-api-url http://localhost:8081
    ulimits:
      nofile:
        soft: 65535
        hard: 65535

  broker2:
    restart: always
    depends_on:
      - rest_api
      - gpu_prove_agent0
      - exec_agent0
      - exec_agent1
      - aux_agent
      - snark_agent
      - redis
      - postgres
    profiles: [broker]
    build:
      context: .
      dockerfile: dockerfiles/broker.dockerfile
    mem_limit: 2G
    cpus: 2
    stop_grace_period: 3h
    volumes:
      - type: bind
        source: ./broker2.toml
        target: /app/broker.toml
      - broker2-data:/db/
    network_mode: host
    env_file:
      - .env.eth-sepolia
    environment:
      RUST_LOG: ${RUST_LOG:-info,broker=debug,boundless_market=debug}
    entrypoint: /app/broker --db-url 'sqlite:///db/broker2.db' --set-verifier-address ${SET_VERIFIER_ADDRESS} --boundless-market-address ${BOUNDLESS_MARKET_ADDRESS} --config-file /app/broker.toml --bento-api-url http://localhost:8081
    ulimits:
      nofile:
        soft: 65535
        hard: 65535

volumes:
  redis-data:
  postgres-data:
  minio-data:
  grafana-data:
  broker-data:
  broker2-data:
```

### Modify `broker.toml`
Each broker instance requires separate `broker.toml` files (e.g., `broker.toml`, `broker2.toml`, etc.)

You can create the new broker config file that the second broker will use:
```bash
# Copy from an existing broker config file
cp broker.toml broker2.toml
 
# Or creating one from a fresh template
cp broker-template.toml broker2.toml
```
Then, modify configuation values for each network, keeping the following in mind:

* The `peak_prove_khz` setting is shared across all brokers.
  * For example, if you have benchmarked your broker to be able to prove at `500kHz`, the values in each config should not sum up to be more than `500kHz`.
  * For instance: `broker.toml`: `peak_prove_khz = 250` & `broker2.toml`: `peak_prove_khz = 250`

* `max_concurrent_preflights` setting limits the number of pricing tasks (preflight executions) a broker can run simultaneously. The total `max_concurrent_preflights` across all brokers (for all networks) should be less than or equal to the number of `exec_agent` services in your `compose.yml`
  * For instance: If you have two `exec_agent` services (`exec_agent0` and `exec_agent1`). Thus, the sum of `max_concurrent_preflights` across `broker` and `broker2` should not exceed `2`.

* `max_concurrent_proofs`
  * Unlike `peak_prove_khz`, the `max_concurrent_proofs` setting is specific to each broker and not shared. It controls the maximum number of proof generation tasks a single broker can process simultaneously.
  * For instance: With only one GPU, your cluster can typically handle only one proof at a time, as proof generation is GPU-intensive. So you'd better to set `max_concurrent_proofs = 1`

 * `lockin_priority_gas`: Make sure you configure the gwei for according to each network

---

# Safe Update or Stop Prover
### 1. Check locked orders
Ensure either through the `broker` logs or [through indexer page of your prover](https://explorer.beboundless.xyz/provers/) that your broker does not have any incomplete locked orders before stopping or update, othervise you might get slashed for your staked assets.

* Optionally to not accept more order requests by your prover temporarily, you can set `max_concurrent_proofs` to `0`, wait for `locked` orders to be `fulfilled`, then go through the next step to stop the node.

### 2. Stop the broker and optionally clean the database
```bash
# Optional, no need if you don't want to upgrade the node's repository
just broker clean
 
# Or stop the broker without cleaning volumes
just broker down
```

### 3. Update to the new version
See [releases](https://github.com/boundless-xyz/boundless/releases) for latest tag to use.
```bash
git checkout <new_version_tag>
# Example: git checkout v0.10.0
```

### 4. Start the broker with the new version
```bash
just broker
```

---

### Network Configurations Method 2: .env files
I **recommend** to go through **Method 1** and skip this step to [Deposit Stake](#deposit-stake), otherwise you can follow method 2 by going here

* There are three `.env` files with the official configurations of each network (`.env.base`, `.env.base-sepolia`, `.env.eth-sepolia`).

### Base Mainnet
* In this step I modify `.env.base`, you can replace it with any of above (Sepolia networks).
* Currently, Base mainnet has very low demand of orders, you may want to go for Base Sepolia by modifying `.env.base-sepolia` or ETH Sepolia by modifying `.env.eth-sepolia`

* Configure `.env.base` file:
```bash
nano .env.base
```
Add the following variables to the `.env.base`.
* `export RPC_URL=""`:
  * RPC has to be between `""`
* `export PRIVATE_KEY=`: Add your EVM wallet private key

![image](https://github.com/user-attachments/assets/3b41c3b7-8f79-4067-9117-41ac68b41946)

* Inject `.env.base` to prover:
```bash
source .env.base
```
* After each terminal close or before any prover startup, you have to run this to inject the network before running `broker` or doing `Deposit` commands (both in next steps).

### Optional: `.env.broker` with custom enviroment
`.env.broker` is a custom environment file same as previous `.env` files but with more options to configure, you can also use it but you have to refer to [Deployments](https://docs.beboundless.xyz/developers/smart-contracts/deployments) page to replace contract addresses of each network.
* I recommend to bypass using it, since you may want to switch between network sometimes. It's easier to swap among those above preserved .env files.

* Create `.env.broker`:
```bash
cp .env.broker-template .env.broker
```

* Configure `.env.broker` file:
```bash
nano .env.broker
```
Add the following variables to the `.env.broker`.
* `export RPC_URL=""`: To get Base network rpc url, Use third-parties .e.g Alchemy or paid ones.
  * RPC has to be between `""`
* `export PRIVATE_KEY=`: Add your EVM wallet private key
* Find the value of following variables [here](https://docs.beboundless.xyz/developers/smart-contracts/deployments):
  * `export BOUNDLESS_MARKET_ADDRESS=`
  * `export SET_VERIFIER_ADDRESS=`
  * `export VERIFIER_ADDRESS=` (add it to .env manually)
  * `export ORDER_STREAM_URL=`
 
* Inject `.env.broker` changes to prover:
```
source .env.broker
```
  * After each terminal close, you have to run this to inject the network before running `broker` or doing `Deposit` commands (both in next steps).

---

# Debugging
## Error: Too many open files (os error 24)
During the build process of `just broker`, you might endup to `Too many open files (os error 24)` error.

### Fix:
```
nano /etc/security/limits.conf
```
* Add:
```
* soft nofile 65535
* hard nofile 65535
```

```
nano /lib/systemd/system/docker.service
```
* Add or modify the following under `[Service]` section.
```
LimitNOFILE=65535
```

```
systemctl daemon-reload
systemctl restart docker
```

* Now restart terminal and rerun your **inject network** command, then run `just broker`


## Getting tens of `Locked` orders on prover's [explorer](https://explorer.beboundless.xyz/)
* It's due to RPC issues, check your logs.
* You can increase `txn_timeout = 45` in `broker.toml` file to increase the seconds of transactions confirmations.






