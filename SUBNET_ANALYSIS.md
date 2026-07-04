================================================================================
                     XDC SUBNET: COMPLETE ANALYSIS REPORT
================================================================================
Location: /home/shalaka/subnet/
Generated: June 24, 2026
Network: xdcSubnet (Chain ID 7670)

================================================================================
1. WHAT IS THIS?
================================================================================

This is a 3-node XDC Blockchain Subnet running locally via Docker. It is a
private/permissioned blockchain network, NOT a public chain like Ethereum mainnet.

  XDC    = XinFin Digital Contract (EVM-compatible blockchain)
  Subnet = A private sidechain / permissioned network for enterprise use

Think of it as a "mini Ethereum" that only YOU control, with 3 validator nodes
that take turns creating blocks every 2 seconds.

--------------------------------------------------------------------------------
KEY PARAMETERS
--------------------------------------------------------------------------------
  Chain ID:        7670
  Network Name:    xdcSubnet
  Consensus:       XDPoS v2 (XinFin Delegated Proof of Stake)
  Block Time:      2 seconds
  Epoch Length:    900 blocks
  Block Reward:    171 XDC per block
  Gap:             450 blocks
  Max Masternodes: 108
  Current Status:  ACTIVE (mining blocks continuously)

================================================================================
2. CONSENSUS: XDPoS v2 (XinFin Delegated Proof of Stake)
================================================================================

XDPoS v2 is a Byzantine Fault Tolerant (BFT) consensus mechanism. It is NOT
Proof of Work (no mining rigs, no electricity waste). It is also NOT simple
Proof of Stake (no slashing, no random selection).

--------------------------------------------------------------------------------
HOW XDPoS v2 WORKS
--------------------------------------------------------------------------------

  1. VALIDATOR SET (MASTERNODES)
     - Only pre-approved nodes can create blocks.
     - In this subnet: exactly 3 masternodes.
     - These are defined in genesis.json extraData and the XDCValidator
       smart contract (address 0x000...0088).

  2. ROUND-ROBIN BLOCK PRODUCTION
     - Masternodes take turns creating blocks in a fixed order.
     - Each node gets a "turn" every 2 seconds.
     - The "whosTurn" field in logs shows which node should mine next.

  3. QUORUM CERTIFICATES (QC)
     - After a block is proposed, other masternodes vote on it.
     - A block is FINALIZED when it receives a Quorum Certificate (QC).
     - QC requires at least 2 out of 3 votes (certificateThreshold = 0.667).
     - This is the "BFT" part: the network tolerates 1 faulty node.

  4. TIMEOUT MECHANISM
     - If the designated masternode fails to produce a block within 10 seconds
       (timeoutPeriod), other nodes send a "timeout" message.
     - After 3 timeout messages (timeoutSyncThreshold), a new round starts
       and the next masternode takes over.

  5. EPOCHS
     - An epoch = 900 blocks.
     - At each epoch boundary, the validator set can be re-evaluated.
     - Reward distribution happens at epoch checkpoints (rewardCheckpoint=900).

--------------------------------------------------------------------------------
XDPoS v2 vs OTHER CONSENSUS MECHANISMS
--------------------------------------------------------------------------------

  +----------------+----------------+----------------+----------------+
  | Feature        | Proof of Work  | Proof of Stake | XDPoS v2       |
  +----------------+----------------+----------------+----------------+
  | Energy Use     | Very High      | Low            | Very Low       |
  | Block Time     | ~12 sec (ETH)  | ~12 sec (ETH)  | ~2 sec (XDC)   |
  | Finality       | Probabilistic  | Probabilistic  | Instant (BFT)  |
  | Validators     | Anyone (open)  | Stake-weighted | Pre-approved   |
  | Fault Tolerance| None           | Economic       | 1/3 BFT        |
  | Best For       | Public chains  | Public chains  | Enterprise     |
  +----------------+----------------+----------------+----------------+

--------------------------------------------------------------------------------
WHY THE LOGS SAY "parent hash and QC hash does not match"
--------------------------------------------------------------------------------

This is a NORMAL and EXPECTED warning in XDPoS v2. It happens during validator
rotation when a node receives a block proposal and the Quorum Certificate (QC)
references a slightly different parent hash due to network timing. The node
still accepts the block if the QC is valid. You see this because the 3 nodes
are competing slightly and reorganizing ("Reorg" in logs) to agree on the canonical
chain.

================================================================================
3. HOW NODES ARE MINING (THE FLOW)
================================================================================

--------------------------------------------------------------------------------
STEP-BY-STEP BLOCK PRODUCTION FLOW
--------------------------------------------------------------------------------

  STEP 1: BOOTSTRAP
  -----------------
  - Bootnode starts first (192.168.25.54:20301).
  - It does NOT mine blocks. It only helps other nodes discover each other.
  - All masternodes connect to the bootnode to find peers.

  STEP 2: GENESIS INITIALIZATION
  -------------------------------
  - Each masternode reads genesis.json on startup.
  - genesis.json contains:
      * Chain ID (7670)
      * Initial validator set (in extraData)
      * Pre-deployed smart contracts (XDCValidator, XDCBlockReward, etc.)
      * Pre-allocated balances (Owner, Foundation, etc.)
  - All nodes must agree on the EXACT same genesis block.

  STEP 3: PEER DISCOVERY
  -----------------------
  - Masternode1 connects to bootnode, learns about Masternode2 and 3.
  - Masternode2 connects to bootnode, learns about Masternode1 and 3.
  - Masternode3 connects to bootnode, learns about Masternode1 and 2.
  - Result: fully connected mesh (each node sees 2 peers).

  STEP 4: CONSENSUS ROUND STARTS
  -------------------------------
  - The network determines "whose turn" it is to mine the next block.
  - Deterministic round-robin based on block number and validator set order.
  - Example from logs: whosTurn=xdc60C8D53B7b788dAEb3a461Ec63D07D4023D99C0D
    This means Masternode2 (key2) is designated to produce the next block.

  STEP 5: BLOCK PROPOSAL
  ----------------------
  - The designated masternode:
      a) Collects pending transactions from its mempool.
      b) Creates a new block with:
           - Parent hash (previous block)
           - Transactions
           - Timestamp
           - Quorum Certificate (QC) from previous round
           - Signature from the proposer
      c) Broadcasts the block to peers.

  STEP 6: VOTING (QC FORMATION)
  -----------------------------
  - Other masternodes receive the proposed block.
  - They validate it (check parent hash, transactions, signature, turn).
  - If valid, they send a VOTE message back to the proposer.
  - Once the proposer collects >= 2 votes (67% threshold), it forms a QC.

  STEP 7: BLOCK FINALIZATION
  ---------------------------
  - The block with its QC is now FINAL.
  - It cannot be reversed (unlike PoW where you need "confirmations").
  - All nodes append it to their local blockchain database (xdcchain*/XDC).

  STEP 8: REWARD DISTRIBUTION
  ---------------------------
  - Block reward (171 XDC) is minted.
  - Rewards go to the masternode that proposed the block.
  - At epoch boundaries (every 900 blocks), rewards are distributed according
    to the XDCBlockReward smart contract rules.

  STEP 9: NEXT ROUND
  ------------------
  - The next masternode in the round-robin order gets its turn.
  - Repeat from Step 4.

--------------------------------------------------------------------------------
THE ACTUAL MINING COMMAND
--------------------------------------------------------------------------------

Each masternode runs the XDC client (a modified Go-Ethereum/Geth) with flags:

  --mine                (enable block production)
  --syncmode full       (full sync, not light)
  --gcmode archive      (keep all historical state)
  --networkid 7670      (custom chain ID)
  --bootnodes <enode>   (connect to bootnode for peer discovery)

The "mining" here is NOT solving cryptographic puzzles (no hash rate). It is
simply "signing and proposing a block" when it is your turn.

================================================================================
4. NETWORK ARCHITECTURE
================================================================================

  Docker Network: 192.168.25.0/24 (docker_net)

  +------------------+     +------------------+     +------------------+
  |  Masternode 1    |<--->|  Masternode 2    |<--->|  Masternode 3    |
  |  Validator       |     |  Validator       |     |  Validator       |
  |  .51:20303       |     |  .52:20304       |     |  .53:20305       |
  |  RPC: 8545       |     |  RPC: 8546       |     |  RPC: 8547       |
  |  WS: 9555        |     |  WS: 9556        |     |  WS: 9557        |
  |  key1 (0x994e..) |     |  key2 (0x60C8..) |     |  key3 (0x5635..) |
  +--------+---------+     +--------+---------+     +--------+---------+
           |                        |                        |
           |                        |                        |
           +------------------------+--------+
                                           |
                                +----------v-----------+
                                |      Bootnode        |
                                |  .54:20301           |
                                |  Peer Discovery Only |
                                |  (Does NOT mine)     |
                                +----------------------+

--------------------------------------------------------------------------------
NODE IDENTITIES
--------------------------------------------------------------------------------

  Masternode 1
    Address:    0x994e38673C40B1F21B087dAFd04AcBBD6bcd970B
    Private:    0xeeddecec51a10059b497301faf089af400075dedbad2d0309767ea4b4618fcde
    RPC:        http://localhost:8545
    Data Dir:   generated/xdcchain1/

  Masternode 2
    Address:    0x60C8D53B7b788dAEb3a461Ec63D07D4023D99C0D
    Private:    0x93ec6fc4e693c1c8d6e5149de2ce7a89244ae2e629fb64af3d2ed2c4baca10bc
    RPC:        http://localhost:8546
    Data Dir:   generated/xdcchain2/

  Masternode 3
    Address:    0x56353Ac7eD5aDa864f411c9719A2Cc416d408FbE
    Private:    0xe9c235075d4af6ca6c7245a6b5e5b4b43a4d1b2f7cabff272732abb7315e6614
    RPC:        http://localhost:8547
    Data Dir:   generated/xdcchain3/

  Owner
    Address:    0x809B71A615F1118D943e9424C24CDF7922c29bc8
    Private:    0xd6763621556b1182214045458b9c9a252ed74c96afec3992cccaf3fbe36bec24
    Role:       Network owner, can add/remove validators

  Foundation
    Address:    0x16aE7a8d946baA913349D1Daa7616966C8BbF8F2
    Private:    0x2de65d5f62a7319714401e3bbd0959465ad5363cd287bc193cc720f70e046de2
    Role:       Receives foundation rewards

================================================================================
5. SMART CONTRACTS IN GENESIS
================================================================================

The genesis block pre-deploys several system smart contracts at fixed addresses:

  0x000...0068  - XDCValidator
                  Manages the validator set, staking, voting, penalties.
                  This is the HEART of XDPoS consensus.

  0x000...0088  - XDCBlockReward
                  Handles block reward distribution to validators.

  0x000...0089  - Randomness contract
                  Provides on-chain randomness for certain operations.

  0x000...0090  - XDCValidator (v2 / additional logic)

  0x000...0099  - Pre-funded account with large balance

These contracts are written in Solidity and their bytecode is embedded in the
"alloc" section of genesis.json. They are NOT deployed by users; they exist
from block 0.

================================================================================
6. FOLDER STRUCTURE
================================================================================

  ~/subnet/
  |-- start_xdpos.sh              # Script to launch the generator UI
  |-- generated/                  # ALL network data lives here
      |-- docker-compose.yml       # Container definitions
      |-- docker-up.sh             # START the network
      |-- docker-down.sh           # STOP the network
      |-- gen.env                  # Basic network config
      |-- genesis_input.yml        # Human-readable genesis input
      |-- genesis.json             # Machine-readable genesis block
      |-- keys.json                # ALL PRIVATE KEYS (SECRET!)
      |-- masternode1.env        # Node 1 config
      |-- masternode2.env        # Node 2 config
      |-- masternode3.env        # Node 3 config
      |-- bootnodes.list           # Bootnode enode address
      |-- bootnodes/               # Bootnode data
      |-- scripts/
      |   |-- check-mining.sh      # Verify blocks are being mined
      |   |-- check-peer.sh        # Verify peer connections
      |-- xdcchain1/               # Node 1 blockchain data
      |   |-- XDC/                 # Main chain database
      |   |-- XDCx/                # Light indexing database
      |   |-- keystore/            # Encrypted wallet files
      |   |-- XDC.ipc             # Unix socket for local API
      |   |-- xdc.log             # Node runtime logs
      |-- xdcchain2/               # Node 2 blockchain data
      |-- xdcchain3/               # Node 3 blockchain data

================================================================================
7. HOW TO INTERACT WITH THE NETWORK
================================================================================

  START THE NETWORK
  -----------------
  cd ~/subnet/generated
  ./docker-up.sh machine1

  STOP THE NETWORK
  ----------------
  cd ~/subnet/generated
  ./docker-down.sh machine1

  CHECK IF MINING
  ---------------
  cd ~/subnet/generated
  ./scripts/check-mining.sh

  CHECK PEER CONNECTIONS
  ----------------------
  cd ~/subnet/generated
  ./scripts/check-peer.sh
  (Should show 2 peers per node)

  QUERY LATEST BLOCK (via RPC)
  ----------------------------
  curl -s http://localhost:8545 -X POST \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

  QUERY BLOCK DETAILS (XDPoS v2)
  ------------------------------
  curl -s http://localhost:8545 -X POST \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"XDPoS_getV2BlockByNumber","params":["latest"],"id":1}'

  GET PEER COUNT
  --------------
  curl -s http://localhost:8545 -X POST \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'

  VIEW NODE LOGS
  --------------
  docker logs -f generated-masternode1-1
  docker logs -f generated-masternode2-1
  docker logs -f generated-masternode3-1

  CHECK RUNNING CONTAINERS
  ------------------------
  docker ps

================================================================================
8. CURRENT STATUS (FROM LOGS)
================================================================================

  Network Status:     ACTIVE and MINING
  Latest Block:       ~8944+ (as of last log entry)
  Block Time:         ~2 seconds
  Peers:              2 per node (fully connected)
  Reorgs:             Occasional (normal for XDPoS v2 during rotation)
  Warnings:           "parent hash and QC hash does not match" (NORMAL)
                      "sendTimeout" (NORMAL during startup/rotation)

  The network has been running continuously since June 23, producing thousands
  of blocks. All 3 masternodes are participating in consensus.

================================================================================
9. IMPORTANT SECURITY NOTES
================================================================================

  1. PRIVATE KEYS
     The file keys.json contains ALL private keys in PLAIN TEXT.
     In a production environment, these MUST be stored securely (HSM, vault,
     encrypted keystore). NEVER commit this file to Git.

  2. LOCAL ONLY
     This setup binds to localhost (127.0.0.1). The RPC ports (8545-8547) are
     NOT exposed to the internet. If you need remote access, use a reverse proxy
     with authentication (e.g., nginx + JWT).

  3. ARCHIVE MODE
     All nodes run with GC_MODE=archive, meaning they keep ALL historical state.
     This is great for debugging but uses significant disk space over time.

  4. DOCKER PRIVILEGES
     Some data folders are owned by root because Docker runs as root.
     Use sudo for file operations if needed, or adjust Docker user mapping.

================================================================================
10. TROUBLESHOOTING
================================================================================

  Problem: Nodes not connecting
  Solution: Check bootnode is running: docker logs generated-bootnode-1
            Verify bootnodes.list has the correct enode address.

  Problem: No blocks being mined
  Solution: Run ./scripts/check-mining.sh
            Check peer count: ./scripts/check-peer.sh
            Verify all 3 containers are up: docker ps

  Problem: Port conflicts
  Solution: Ensure ports 20303-20305, 8545-8547, 9555-9557 are free.
            Check: sudo lsof -i :8545

  Problem: Permission denied on xdcchain*/ folders
  Solution: sudo chown -R $(whoami):$(whoami) generated/xdcchain*

  Problem: "parent hash and QC hash does not match" warnings
  Solution: This is NORMAL. No action needed. It is part of XDPoS v2 consensus.

  Problem: Chain forked / nodes disagree
  Solution: Stop all nodes, delete xdcchain*/XDC and xdcchain*/XDCx folders
            (keep keystore/), then restart. They will re-sync from genesis.

================================================================================
11. GLOSSARY
================================================================================

  XDC          XinFin Digital Contract - the blockchain platform
  XDPoS        XinFin Delegated Proof of Stake - consensus algorithm
  XDPoS v2     Second generation with BFT and Quorum Certificates
  Masternode   A validator node authorized to produce blocks
  Bootnode     A helper node for peer discovery (does not mine)
  Genesis      The first block of the blockchain (block 0)
  Epoch        A period of 900 blocks; validator set can change
  QC           Quorum Certificate - proof that >=2/3 validators agree
  BFT          Byzantine Fault Tolerance - tolerates up to 1/3 faulty nodes
  Reorg        Reorganization - when nodes switch to a different chain tip
  Enode        Ethereum node identifier (public key + IP + port)
  ExtraData    Field in block header containing validator signatures
  Turn         Which masternode's turn it is to propose the next block

================================================================================
                         END OF REPORT
================================================================================
