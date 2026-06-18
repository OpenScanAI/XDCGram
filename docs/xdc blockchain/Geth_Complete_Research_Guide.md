# Geth Private Blockchain — Complete Research & Implementation Guide

> **Chain ID:** 12345 | **Geth Version:** v1.13.15 | **Consensus:** Clique PoA  
> **Coverage:** All 5 research areas — deep dive theory + working code

---

## Table of Contents

1. [Geth Version Analysis](#1-geth-version-analysis)
2. [UPI & ENS — Human-Readable Identities](#2-upi--ens--human-readable-identities)
3. [Transaction & Visibility Testing](#3-transaction--visibility-testing)
4. [Bot Integration & Testing](#4-bot-integration--testing)
5. [Multi-Client Node Testing](#5-multi-client-node-testing)

---

# 1. Geth Version Analysis

## 1.1 What is Geth and Why Versions Matter

Geth (Go Ethereum) is software written in the Go programming language that implements the Ethereum protocol. Think of the Ethereum protocol as a set of rules — like the rules of chess. Geth is one program that plays by those rules. Just like chess software gets updated, Geth gets updated too.

Each version update can:
- Add new features
- Remove old/unsafe features
- Change how you configure and start the node
- Change which APIs are available

This matters enormously for private blockchain deployment because **code written for Geth v1.10 may not work at all on Geth v1.17**.

---

## 1.2 The Ethereum Upgrade Timeline (Context)

To understand why Geth changed so much between versions, you need to understand Ethereum's own history:

```
2015 — Ethereum Mainnet launches (Proof of Work)
2020 — Beacon Chain launches (Proof of Stake chain runs in parallel)
2022 — The Merge: Ethereum switches from PoW to PoS permanently
2023+ — Post-merge upgrades (Shanghai, Cancun, etc.)
```

Before The Merge, one client (like Geth) handled everything:
- Block production
- Transaction processing
- Networking
- JSON-RPC API

After The Merge, Ethereum split into two layers:
- **Execution Layer (EL)** — Geth handles transactions and state (the "what happened")
- **Consensus Layer (CL)** — A separate client (like Prysm, Lighthouse) handles block finality (the "which chain wins")

This split is the single biggest reason Geth changed so dramatically between versions.

---

## 1.3 Version-by-Version Breakdown

### Geth v1.9.x (2019–2020) — The Classic Era

This was the pre-Merge, pre-split era. Everything ran in one process.

**Starting a node looked like this:**
```bash
geth --rpc --rpcaddr 127.0.0.1 --rpcport 8545 --rpcapi eth,web3,net,personal
```

Key characteristics:
- `--rpc` flag enabled HTTP JSON-RPC (now renamed to `--http`)
- `--rpcaddr`, `--rpcport`, `--rpcapi` were the flag names (all later renamed with `--http.*` prefix)
- WebSocket used `--ws`, `--wsaddr`, `--wsport`, `--wsapi`
- `personal` API was fully available — you could create accounts, unlock them, sign transactions all over HTTP
- Mining worked with `--mine` and `--miner.threads`
- Clique PoA fully supported

---

### Geth v1.10.x (2021) — Transition Begins

This version started preparing for The Merge. Most things still worked the same but deprecation warnings appeared.

**Key change:** The `--rpc` flag was deprecated in favour of `--http`. Both worked but `--rpc` showed a warning.

```bash
# Old way (deprecated, showed warning)
geth --rpc --rpcaddr 127.0.0.1

# New way
geth --http --http.addr 127.0.0.1
```

**Why the rename?** Because `--rpc` was ambiguous. Ethereum now had multiple RPC types — HTTP, WebSocket, IPC, and later the Engine API. The rename made it clear which transport you were configuring.

---

### Geth v1.11.x — v1.13.x (2022–2023) — The Post-Merge Era

This is the version you are running (v1.13.15). After The Merge:

- The `personal` API was deprecated over HTTP for security reasons (you can still use it over IPC)
- The `--allow-insecure-unlock` flag was added to override the HTTP restriction on account unlocking
- Clique PoA still worked — this is the last era where Clique was supported
- The Engine API was introduced (port 8551 with JWT authentication) for communication between Geth and a consensus client

**Your startup command explained flag by flag:**
```bash
geth \
  --datadir XDC-T \               # Where chain data is stored
  --networkid 12345 \             # Your private network ID
  --http \                        # Enable HTTP JSON-RPC (was --rpc in old versions)
  --http.addr 127.0.0.1 \         # Only accept local connections (was --rpcaddr)
  --http.port 8545 \              # Port number (was --rpcport)
  --http.api admin,eth,web3,net,personal,miner,txpool \ # APIs to expose (was --rpcapi)
  --http.corsdomain "*" \         # Allow any website to call your RPC (CORS)
  --nodiscover \                  # Don't try to find peers on the internet
  --unlock 0xYOUR_ADDRESS \       # Auto-unlock account on startup
  --allow-insecure-unlock \       # Allow unlock over HTTP (needed since v1.11+)
  --miner.etherbase 0xYOUR_ADDRESS \ # Address that receives block rewards
  --mine \                        # Start sealing blocks (Clique PoA)
  console                         # Open interactive JavaScript console
```

---

### Geth v1.14.x+ — v1.17.x (2024+) — Clique Removed

This is where things break for private network developers. Starting in v1.14, Clique PoA was removed from Geth.

**Why was Clique removed?**

The Ethereum team's reasoning:
1. Ethereum mainnet no longer uses PoA — it uses PoS
2. Clique was considered a "transitional" mechanism that served its purpose
3. Private networks that need PoA should use dedicated tools like Hyperledger Besu or run a proper PoS devnet
4. Maintaining Clique code added complexity with no benefit to mainnet users

**What this means for you:**
```bash
# On Geth v1.17, starting with Clique genesis fails:
geth init --datadir XDC-T genesis.json
# ERROR: Geth only supports PoS networks
# The genesis file needs proof-of-stake configuration
```

**What else changed in v1.17:**

| Feature | v1.13.15 | v1.17+ |
|---|---|---|
| Clique PoA | ✅ Supported | ❌ Removed |
| `personal` API | ✅ Works (with flag) | ❌ Removed entirely |
| `--allow-insecure-unlock` | ✅ Works | ❌ Flag removed |
| `eth_accounts` (returns local accounts) | ✅ Works | ⚠️ Empty (no local signer) |
| `--mine` flag | ✅ Works | ❌ No longer meaningful without CL |
| Engine API (port 8551) | Optional | Required for PoS |
| JWT authentication | Optional | Required |

---

## 1.4 The `--rpc` → `--http` Flag Migration

This is one of the most common sources of confusion when reading old Ethereum tutorials. Here is the complete mapping:

### HTTP RPC Flags

| Old Flag (v1.9 and earlier) | New Flag (v1.10+) | What It Does |
|---|---|---|
| `--rpc` | `--http` | Enable HTTP RPC server |
| `--rpcaddr 0.0.0.0` | `--http.addr 0.0.0.0` | Bind address |
| `--rpcport 8545` | `--http.port 8545` | Port number |
| `--rpcapi eth,web3` | `--http.api eth,web3` | APIs to enable |
| `--rpccorsdomain "*"` | `--http.corsdomain "*"` | CORS origins |
| `--rpcvhosts "*"` | `--http.vhosts "*"` | Virtual host filtering |

### WebSocket Flags (unchanged names)

| Flag | What It Does |
|---|---|
| `--ws` | Enable WebSocket server |
| `--ws.addr` | Bind address |
| `--ws.port 8546` | Port (default 8546) |
| `--ws.api` | APIs to enable |

### IPC (always worked, no change needed)

IPC is a local socket file. On Windows it's a named pipe (`\\.\pipe\geth.ipc`), on Linux it's a Unix socket file (`XDC-T/geth.ipc`). No flags needed — it's enabled by default.

---

## 1.5 The `personal` API — Why It Was Restricted Then Removed

### What `personal` does

The `personal` API is a set of RPC methods for managing accounts directly inside Geth:

```
personal_newAccount        — create a new account
personal_unlockAccount     — unlock an account with password
personal_lockAccount       — lock an account
personal_sign              — sign data with an account
personal_sendTransaction   — sign and send in one step
personal_listAccounts      — list all accounts
```

### Why it became dangerous over HTTP

When you expose `personal` over HTTP with `--http.api personal`, anyone who can reach port 8545 can:
1. Call `personal_unlockAccount` with a password guess
2. If they succeed, call `personal_sendTransaction` to drain funds
3. On a local-only setup (`--http.addr 127.0.0.1`) this is low risk, but on a server exposed to the internet it's catastrophic

### The security progression:

```
v1.9  — personal works over HTTP by default, no warning
v1.10 — personal over HTTP shows deprecation warning  
v1.11 — personal over HTTP blocked unless you add --allow-insecure-unlock
v1.17 — personal API removed entirely
```

### How to replace `personal` in v1.17+

Since you must use v1.13.15 for Clique, this is future knowledge. But for completeness:
- Use `eth_signTransaction` with an external signer (like `clef`)
- Use `clef` — Geth's dedicated account management daemon
- Use a keystore library in your application (ethers.js, web3.js handle this)

---

## 1.6 The Engine API and JWT Authentication (v1.13+ for PoS)

Even though you are using Clique and don't need this, understanding it is important for future work.

After The Merge, Geth needs to communicate with a consensus client (Prysm, Lighthouse, etc.). They communicate over a special API called the Engine API on port 8551.

**Why port 8551 instead of 8545?**

Because the Engine API must never be publicly accessible — it controls block production. It uses JWT (JSON Web Token) authentication to ensure only the consensus client can talk to it.

```bash
# Generate JWT secret
openssl rand -hex 32 > jwt.hex

# Start Geth with Engine API (PoS setup, NOT for your private chain)
geth \
  --http --http.port 8545 \
  --authrpc.addr 127.0.0.1 \
  --authrpc.port 8551 \
  --authrpc.jwtsecret ./jwt.hex
```

The consensus client also reads the same `jwt.hex` file and uses it to authenticate its requests to Geth.

---

## 1.7 Impact Summary for Your Private Network

| Decision | Reason |
|---|---|
| Stay on Geth v1.13.15 | Last version with Clique + personal API |
| Use `--http` not `--rpc` | `--rpc` was removed, `--http` is the correct flag |
| Use `--allow-insecure-unlock` | Required since v1.11 to unlock accounts over HTTP |
| Use IPC for sensitive operations | Full API access, no security restrictions |
| Don't upgrade to v1.14+ | Clique removed, will break your genesis |

---

# 2. UPI & ENS — Human-Readable Identities

## 2.1 The Problem: Wallet Addresses Are Unusable

A typical Ethereum wallet address looks like this:

```
0x9Be2613FB76AcA8cc3530Cf7E402c384e28A49E7
```

This is 42 characters of hexadecimal. As a payment identifier, it is:
- Nearly impossible to memorize
- Easy to mistype (one wrong character = lost funds)
- Impossible to associate with a person without external reference
- Visually indistinguishable between different addresses

Compare that to a UPI ID:

```
rahul@okicici
```

This is readable, memorable, and clearly identifies a person and their bank. This gap between blockchain addresses and real-world usability is one of the biggest barriers to mainstream adoption.

---

## 2.2 How UPI Works — Architecture Deep Dive

UPI (Unified Payments Interface) was built by NPCI (National Payments Corporation of India) and launched in 2016. It is one of the most successful payment systems ever built.

### The Core Architecture

```
┌─────────────────────────────────────────────────────┐
│                    UPI System                        │
│                                                      │
│  User A (rahul@okicici)                              │
│       │                                              │
│       ▼                                              │
│  ICICI Bank (PSP — Payment Service Provider)         │
│       │                                              │
│       │ Routes via VPA lookup                        │
│       ▼                                              │
│  NPCI Central Switch                                 │
│       │                                              │
│       │ Resolves VPA → account number               │
│       ▼                                              │
│  SBI Bank (Recipient's PSP)                          │
│       │                                              │
│       ▼                                              │
│  User B (priya@sbi)                                  │
└─────────────────────────────────────────────────────┘
```

### Key Concepts

**VPA (Virtual Payment Address)** — This is the UPI ID like `rahul@okicici`. It is a human-readable alias that maps to an actual bank account number + IFSC code. The mapping is stored in NPCI's central registry.

**PSP (Payment Service Provider)** — Apps like PhonePe, Google Pay, Paytm are PSPs. They provide the interface but the actual payment routing goes through NPCI.

**The handle (`@okicici`, `@ybl`, `@paytm`)** — The part after @ identifies which PSP/bank issued the VPA. It acts like a namespace.

### What Happens When You Pay rahul@okicici

1. You enter `rahul@okicici` in your payment app
2. Your PSP sends a VPA resolution request to NPCI
3. NPCI looks up `rahul@okicici` in its registry → finds the mapped account (account number + IFSC)
4. NPCI routes the payment instruction to ICICI
5. ICICI credits Rahul's account
6. NPCI sends confirmation back
7. Your PSP shows "Payment successful"

**Critically: the sender never sees Rahul's actual account number or IFSC.** The VPA is the only identifier used in the transaction flow.

---

## 2.3 How ENS Works — Architecture Deep Dive

ENS (Ethereum Name Service) was deployed on Ethereum mainnet in 2017. It is a set of smart contracts that map human-readable names to Ethereum addresses (and other data).

### The Core Contracts

ENS consists of three main smart contracts:

```
┌──────────────────────────────────────────────────────┐
│                  ENS Architecture                     │
│                                                       │
│  1. ENS Registry                                      │
│     • Single contract that owns all names             │
│     • Maps name hash → owner + resolver               │
│     • Lives at 0x00000000000C2E074eC69A0dFb2997BA6C7d│
│                                                       │
│  2. Resolver Contract                                  │
│     • Stores the actual data (address, IPFS, email)   │
│     • Called by apps to look up what a name points to │
│                                                       │
│  3. Registrar Contract                                 │
│     • Handles registration of .eth names              │
│     • Runs the auction/rental process                 │
└──────────────────────────────────────────────────────┘
```

### Namehash — How Names Become Contract Keys

ENS doesn't store the string `"vitalik.eth"` directly. Instead it uses a mathematical function called **namehash** to convert names into 32-byte hashes.

```javascript
// Conceptual namehash algorithm
namehash("") = 0x0000000000000000000000000000000000000000000000000000000000000000
namehash("eth") = keccak256(namehash("") + keccak256("eth"))
namehash("vitalik.eth") = keccak256(namehash("eth") + keccak256("vitalik"))
```

This is recursive. Every label (part separated by dots) gets hashed and combined with the parent. This makes ENS infinitely extensible — `sub.vitalik.eth` just adds another level.

### What Happens When an App Resolves `vitalik.eth`

```
App asks: "What address is vitalik.eth?"
     │
     ▼
1. App calls ENS Registry: getResolver(namehash("vitalik.eth"))
     │
     ▼
2. Registry returns: address of the Resolver contract
     │
     ▼
3. App calls Resolver: addr(namehash("vitalik.eth"))
     │
     ▼
4. Resolver returns: 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
     │
     ▼
5. App uses that address for the transaction
```

### Reverse Resolution

ENS also supports reverse resolution — mapping from address back to name. This is how apps show `vitalik.eth` in the UI instead of the raw address.

```
Address: 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
     │
     ▼
Reverse Registry: reverseNode = namehash("d8da...96045.addr.reverse")
     │
     ▼
Resolver for that node: name() → "vitalik.eth"
```

---

## 2.4 UPI vs ENS vs Wallet Addresses — Full Comparison

| Dimension | Wallet Address | UPI ID | ENS Name |
|---|---|---|---|
| Format | `0x9Be2...A49E7` | `rahul@okicici` | `alice.xdc` |
| Length | 42 chars hex | 10–30 chars | Variable |
| Human-readable | ❌ No | ✅ Yes | ✅ Yes |
| Decentralized | ✅ Yes | ❌ No (NPCI controls) | ✅ Yes (on-chain) |
| Requires trust in 3rd party | ❌ No | ✅ Yes (NPCI + banks) | ❌ No |
| Can be lost/revoked | ❌ No | ✅ Yes (bank can close) | ⚠️ If not renewed |
| Privacy | ⚠️ Pseudonymous | ❌ Linked to identity | ⚠️ Pseudonymous |
| Transaction settlement | Blockchain | Bank-to-bank | Blockchain |
| Reversible payments | ❌ No | ✅ Yes (dispute process) | ❌ No |
| Cost to register | ❌ Free | ❌ Free | 💰 ETH/year rent |
| Works offline | ❌ No | ✅ Limited (USSD) | ❌ No |
| Namespace governed by | No one | NPCI (government body) | ENS DAO |

---

## 2.5 Building a Custom ENS-Like Naming System on Your Private Chain

This is where theory becomes practice. We will write a Solidity smart contract that acts as a simple name registry — mapping names like `alice.xdc` to wallet addresses.

### Step 1 — Understanding the Contract Design

Our registry needs to:
1. Store mappings: name string → wallet address
2. Allow anyone to register a name (if unclaimed)
3. Allow the owner to update where their name points
4. Allow anyone to look up a name
5. Support reverse lookup (address → name)

### Step 2 — Install Solidity Compiler

```bash
# Install solc globally via npm
npm install -g solc

# Verify
solcjs --version
```

### Step 3 — Write the Registry Contract

Create a file called `NameRegistry.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/**
 * NameRegistry — A simple ENS-like naming system for private chains
 *
 * How it works:
 * - Anyone can register a name if it is not taken
 * - The registrant becomes the owner of that name
 * - The owner can point the name to any address
 * - The owner can transfer ownership to another address
 * - Anyone can look up a name to get its resolved address
 * - Anyone can do reverse lookup: address → name
 */
contract NameRegistry {

    // Stores each name's data
    struct NameRecord {
        address owner;       // Who owns/controls this name
        address resolved;    // What address this name points to
        uint256 registeredAt; // Block number when registered
    }

    // name string → record
    mapping(string => NameRecord) private registry;

    // address → primary name (reverse lookup)
    mapping(address => string) private reverseRegistry;

    // Events — emitted when things happen, visible in transaction logs
    event NameRegistered(string name, address owner, address resolved);
    event NameUpdated(string name, address newResolved);
    event OwnershipTransferred(string name, address oldOwner, address newOwner);
    event PrimaryNameSet(address indexed addr, string name);

    // ─────────────────────────────────────────
    // WRITE functions (cost gas, change state)
    // ─────────────────────────────────────────

    /**
     * Register a new name
     * - name: the identifier like "alice" (you add .xdc in the UI)
     * - resolved: the wallet address this name should point to
     *
     * Reverts if name is already taken
     */
    function register(string calldata name, address resolved) external {
        require(bytes(name).length > 0, "Name cannot be empty");
        require(bytes(name).length <= 64, "Name too long (max 64 chars)");
        require(resolved != address(0), "Cannot resolve to zero address");
        require(registry[name].owner == address(0), "Name already registered");

        registry[name] = NameRecord({
            owner: msg.sender,
            resolved: resolved,
            registeredAt: block.number
        });

        emit NameRegistered(name, msg.sender, resolved);
    }

    /**
     * Update where your name points to
     * Only the owner can do this
     */
    function updateResolved(string calldata name, address newResolved) external {
        require(registry[name].owner == msg.sender, "Not the name owner");
        require(newResolved != address(0), "Cannot resolve to zero address");

        registry[name].resolved = newResolved;
        emit NameUpdated(name, newResolved);
    }

    /**
     * Transfer name ownership to another address
     */
    function transferOwnership(string calldata name, address newOwner) external {
        require(registry[name].owner == msg.sender, "Not the name owner");
        require(newOwner != address(0), "Cannot transfer to zero address");

        address oldOwner = registry[name].owner;
        registry[name].owner = newOwner;
        emit OwnershipTransferred(name, oldOwner, newOwner);
    }

    /**
     * Set your primary name (reverse lookup)
     * Lets apps show "alice.xdc" instead of your raw address
     */
    function setPrimaryName(string calldata name) external {
        // Must own the name and it must resolve to your address
        require(registry[name].owner == msg.sender, "Not the name owner");
        reverseRegistry[msg.sender] = name;
        emit PrimaryNameSet(msg.sender, name);
    }

    // ─────────────────────────────────────────
    // READ functions (free, no gas)
    // ─────────────────────────────────────────

    /**
     * Look up a name → returns the resolved wallet address
     * Returns address(0) if name is not registered
     */
    function resolve(string calldata name) external view returns (address) {
        return registry[name].resolved;
    }

    /**
     * Get full record for a name
     */
    function getRecord(string calldata name)
        external view
        returns (address owner, address resolved, uint256 registeredAt)
    {
        NameRecord memory record = registry[name];
        return (record.owner, record.resolved, record.registeredAt);
    }

    /**
     * Check if a name is available
     */
    function isAvailable(string calldata name) external view returns (bool) {
        return registry[name].owner == address(0);
    }

    /**
     * Reverse lookup: address → primary name
     * Returns empty string if no primary name set
     */
    function reverseLookup(address addr) external view returns (string memory) {
        return reverseRegistry[addr];
    }
}
```

### Step 4 — Compile the Contract

```bash
# Compile — produces ABI and bytecode
solcjs --bin --abi NameRegistry.sol --output-dir ./compiled

# You should see two files created:
# compiled/NameRegistry_sol_NameRegistry.abi  — the interface (what functions exist)
# compiled/NameRegistry_sol_NameRegistry.bin  — the bytecode (what gets deployed)
```

**What is ABI?**  
ABI (Application Binary Interface) is a JSON description of every function in the contract — its name, inputs, and outputs. It's how JavaScript (web3.js/ethers.js) knows how to call your contract.

**What is bytecode?**  
The compiled contract as machine code for the Ethereum Virtual Machine (EVM). This is what gets stored on the blockchain when you deploy.

### Step 5 — Deploy the Contract via RPC

With your Geth node running, deploy using this Node.js script. Create `deploy.js`:

```javascript
const fs = require('fs');
const https = require('http');

// Read compiled files
const abi = JSON.parse(fs.readFileSync('./compiled/NameRegistry_sol_NameRegistry.abi', 'utf8'));
const bytecode = '0x' + fs.readFileSync('./compiled/NameRegistry_sol_NameRegistry.bin', 'utf8').trim();

const RPC_URL = 'http://127.0.0.1:8545';
const FROM_ADDRESS = '0xYOUR_ADDRESS_HERE'; // Replace with your account

// Helper: send JSON-RPC request
function rpc(method, params) {
    return new Promise((resolve, reject) => {
        const body = JSON.stringify({ jsonrpc: '2.0', method, params, id: 1 });
        const req = https.request({
            hostname: '127.0.0.1',
            port: 8545,
            path: '/',
            method: 'POST',
            headers: { 'Content-Type': 'application/json' }
        }, (res) => {
            let data = '';
            res.on('data', chunk => data += chunk);
            res.on('end', () => {
                const parsed = JSON.parse(data);
                if (parsed.error) reject(new Error(parsed.error.message));
                else resolve(parsed.result);
            });
        });
        req.on('error', reject);
        req.write(body);
        req.end();
    });
}

async function deploy() {
    console.log('Deploying NameRegistry contract...');

    // Step 1: Estimate gas
    const gasEstimate = await rpc('eth_estimateGas', [{
        from: FROM_ADDRESS,
        data: bytecode
    }]);
    console.log('Gas estimate:', parseInt(gasEstimate, 16));

    // Step 2: Send deployment transaction
    const txHash = await rpc('eth_sendTransaction', [{
        from: FROM_ADDRESS,
        data: bytecode,
        gas: gasEstimate
    }]);
    console.log('Transaction hash:', txHash);

    // Step 3: Wait for receipt (poll every second)
    console.log('Waiting for mining...');
    let receipt = null;
    while (!receipt) {
        await new Promise(r => setTimeout(r, 1000));
        receipt = await rpc('eth_getTransactionReceipt', [txHash]);
    }

    console.log('Contract deployed!');
    console.log('Contract address:', receipt.contractAddress);
    console.log('Block number:', parseInt(receipt.blockNumber, 16));

    // Save contract address for later use
    fs.writeFileSync('./contract_address.txt', receipt.contractAddress);
    console.log('Address saved to contract_address.txt');
}

deploy().catch(console.error);
```

Run it:
```bash
node deploy.js
```

### Step 6 — Interact with the Contract

Create `interact.js`:

```javascript
const fs = require('fs');
const http = require('http');

const CONTRACT_ADDRESS = fs.readFileSync('./contract_address.txt', 'utf8').trim();
const FROM_ADDRESS = '0xYOUR_ADDRESS_HERE';

// ABI function selectors — first 4 bytes of keccak256 of function signature
// You can get these from ethers.js or compute manually
// For simplicity we use ethers.js here

// Install: npm install ethers
const { ethers } = require('ethers');
const abi = JSON.parse(fs.readFileSync('./compiled/NameRegistry_sol_NameRegistry.abi', 'utf8'));

const provider = new ethers.JsonRpcProvider('http://127.0.0.1:8545');
const signer = await provider.getSigner(FROM_ADDRESS);
const contract = new ethers.Contract(CONTRACT_ADDRESS, abi, signer);

async function demo() {
    // Register a name
    console.log('Registering "alice"...');
    const tx = await contract.register('alice', FROM_ADDRESS);
    await tx.wait();
    console.log('Registered! TX:', tx.hash);

    // Resolve the name
    const resolved = await contract.resolve('alice');
    console.log('alice resolves to:', resolved);

    // Check if available
    const available = await contract.isAvailable('bob');
    console.log('bob is available:', available);

    // Set primary name
    const tx2 = await contract.setPrimaryName('alice');
    await tx2.wait();
    console.log('Primary name set!');

    // Reverse lookup
    const primaryName = await contract.reverseLookup(FROM_ADDRESS);
    console.log('Primary name for address:', primaryName);
}

demo().catch(console.error);
```

---

## 2.6 How a Full UPI-Like System Would Work on Your Chain

Combining everything above, here is the architecture for a UPI-like payment system on your private chain:

```
User A wants to pay "alice.xdc"
           │
           ▼
  App calls NameRegistry.resolve("alice")
           │
           ▼
  Contract returns: 0x9Be2...A49E7
           │
           ▼
  App shows: "Sending to alice.xdc (0x9Be2...)"
           │
           ▼
  User confirms
           │
           ▼
  App calls eth_sendTransaction to 0x9Be2...
           │
           ▼
  Transaction mined in ~5 seconds (Clique)
           │
           ▼
  App shows receipt with alice.xdc label
```

The key insight: **the blockchain stores raw addresses, but the app uses the name registry to show human-readable names.** The name is resolved client-side before the transaction is sent.

---

# 3. Transaction & Visibility Testing

## 3.1 What We Are Testing and Why

When a transaction happens on a blockchain, several layers of information become visible:

1. **On-chain data** — permanently stored in the blockchain, visible to anyone with an RPC node
2. **Application-layer data** — what MetaMask or your web app shows the user
3. **Network-layer data** — what IP addresses and metadata the node operator can see

Understanding all three layers is critical for building a payment system, because it determines what privacy users have.

---

## 3.2 Setup — Two Interfaces

We will create two interfaces:
- **Interface A:** MetaMask (browser wallet) — represents User A sending money
- **Interface B:** A custom web page built with ethers.js — represents User B receiving money

### Setting Up MetaMask

1. Install MetaMask browser extension from metamask.io
2. Create a new wallet (or import existing)
3. Add your private network:
   - Open MetaMask → click network dropdown → "Add network" → "Add a network manually"
   - Network Name: `XDC-T Private`
   - RPC URL: `http://127.0.0.1:8545`
   - Chain ID: `12345`
   - Currency Symbol: `ETH`
4. Import your account using its private key:
   - In Geth console: `personal.exportRawKey(eth.accounts[0], "your-password")`
   - In MetaMask: Account menu → "Import account" → paste private key

### Setting Up Interface B (Web Page)

Create a folder called `interface-b/` and create `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>XDC-T Wallet — User B Interface</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/ethers/6.7.0/ethers.umd.min.js"></script>
    <style>
        body { font-family: monospace; max-width: 600px; margin: 40px auto; padding: 20px; }
        .card { border: 1px solid #ccc; padding: 16px; margin: 12px 0; border-radius: 8px; }
        .label { color: #666; font-size: 12px; }
        .value { font-size: 14px; word-break: break-all; }
        button { padding: 8px 16px; background: #333; color: white; border: none;
                 border-radius: 4px; cursor: pointer; margin: 4px; }
        #log { background: #f5f5f5; padding: 12px; height: 200px; overflow-y: auto;
               font-size: 12px; border-radius: 4px; }
    </style>
</head>
<body>
    <h2>XDC-T — User B Interface</h2>

    <div class="card">
        <div class="label">RPC Endpoint</div>
        <div class="value">http://127.0.0.1:8545</div>
    </div>

    <div class="card">
        <div class="label">Account Address</div>
        <div class="value" id="address">Not connected</div>
        <div class="label" style="margin-top:8px">Balance</div>
        <div class="value" id="balance">—</div>
        <div class="label" style="margin-top:8px">Transaction Count (Nonce)</div>
        <div class="value" id="nonce">—</div>
    </div>

    <button onclick="connect()">Connect to Node</button>
    <button onclick="refreshBalance()">Refresh Balance</button>
    <button onclick="watchIncoming()">Watch for Incoming TX</button>

    <div class="card">
        <div class="label">Latest Incoming Transaction</div>
        <div id="latestTx">None yet</div>
    </div>

    <div class="label">Event Log</div>
    <div id="log"></div>

    <script>
        // Configuration — change these to your actual addresses
        const RPC_URL = 'http://127.0.0.1:8545';
        const USER_B_ADDRESS = '0xYOUR_ACCOUNT_1_ADDRESS'; // Second account

        let provider;

        function log(msg) {
            const el = document.getElementById('log');
            const time = new Date().toLocaleTimeString();
            el.innerHTML += `[${time}] ${msg}\n`;
            el.scrollTop = el.scrollHeight;
        }

        async function connect() {
            try {
                provider = new ethers.JsonRpcProvider(RPC_URL);
                const network = await provider.getNetwork();
                log(`Connected. Chain ID: ${network.chainId}`);
                await refreshBalance();
            } catch (err) {
                log('Error: ' + err.message);
            }
        }

        async function refreshBalance() {
            if (!provider) { log('Not connected'); return; }
            try {
                const balance = await provider.getBalance(USER_B_ADDRESS);
                const nonce = await provider.getTransactionCount(USER_B_ADDRESS);
                const blockNumber = await provider.getBlockNumber();

                document.getElementById('address').textContent = USER_B_ADDRESS;
                document.getElementById('balance').textContent =
                    ethers.formatEther(balance) + ' ETH';
                document.getElementById('nonce').textContent = nonce;

                log(`Balance: ${ethers.formatEther(balance)} ETH | Block: ${blockNumber}`);
            } catch (err) {
                log('Error: ' + err.message);
            }
        }

        async function watchIncoming() {
            if (!provider) { log('Not connected'); return; }
            log('Watching for incoming transactions (polling every 3 seconds)...');

            let lastBlock = await provider.getBlockNumber();

            setInterval(async () => {
                try {
                    const currentBlock = await provider.getBlockNumber();
                    if (currentBlock <= lastBlock) return;

                    for (let b = lastBlock + 1; b <= currentBlock; b++) {
                        const block = await provider.getBlock(b, true);
                        if (!block || !block.transactions) continue;

                        for (const tx of block.transactions) {
                            if (tx.to &&
                                tx.to.toLowerCase() === USER_B_ADDRESS.toLowerCase()) {
                                log(`INCOMING TX DETECTED!`);
                                log(`  Hash: ${tx.hash}`);
                                log(`  From: ${tx.from}`);
                                log(`  Amount: ${ethers.formatEther(tx.value)} ETH`);
                                log(`  Block: ${tx.blockNumber}`);
                                log(`  Gas Price: ${ethers.formatUnits(tx.gasPrice, 'gwei')} Gwei`);

                                document.getElementById('latestTx').innerHTML = `
                                    <div class="label">Hash</div>
                                    <div class="value">${tx.hash}</div>
                                    <div class="label">From</div>
                                    <div class="value">${tx.from}</div>
                                    <div class="label">Amount</div>
                                    <div class="value">${ethers.formatEther(tx.value)} ETH</div>
                                    <div class="label">Block</div>
                                    <div class="value">${tx.blockNumber}</div>
                                `;

                                await refreshBalance();
                            }
                        }
                    }
                    lastBlock = currentBlock;
                } catch (err) {
                    log('Poll error: ' + err.message);
                }
            }, 3000);
        }
    </script>
</body>
</html>
```

Open this file directly in your browser (no server needed — it connects directly to the RPC).

---

## 3.3 What Information Is Visible During a Transaction

### On-Chain Visibility (Anyone With RPC Access Can See)

When you send a transaction, these fields are stored permanently on-chain and readable by anyone:

```json
{
  "hash": "0x9fb391...",         // Unique transaction ID
  "from": "0x9Be2...",           // Sender's address — ALWAYS VISIBLE
  "to": "0x7c5b...",             // Recipient's address — ALWAYS VISIBLE
  "value": "0x8AC7230489E80000", // Amount in Wei — ALWAYS VISIBLE
  "gas": "0x5208",               // Gas limit set by sender
  "gasPrice": "0x3b9aca00",      // Gas price paid
  "nonce": "0x1",                // Transaction count for sender
  "blockNumber": "0x5",          // Which block it was in
  "blockHash": "0xabc...",       // Hash of that block
  "transactionIndex": "0x0",     // Position in the block
  "input": "0x"                  // Any extra data (empty for ETH transfer)
}
```

**Key insight:** Every transaction permanently records the sender and recipient address. There is no concept of private transactions in standard Ethereum (privacy coins like Monero or ZK-based systems handle this differently).

### What Is NOT Visible On-Chain

- **IP addresses** — the blockchain stores no networking information
- **Names or identities** — only raw addresses (unless resolved via ENS/NameRegistry)
- **Location data**
- **Device information**

### What the NODE OPERATOR Can See (IP Visibility)

This is an important distinction. While the blockchain doesn't store IP addresses, **the Geth node operator can see the IP address of anyone who submits a transaction directly to their node.**

```
User submits TX → HTTP POST to http://127.0.0.1:8545
                   Geth node logs: IP address in connection log
```

If you run a **public RPC endpoint**, you see the IP of every client that submits transactions to you. On your private network with `--http.addr 127.0.0.1` only localhost can connect, so this is not a concern.

On a public network, this is why people use:
- VPNs before submitting transactions
- Services like Flashbots (submit to miners directly)
- Tor (routes traffic through multiple nodes)

### What MetaMask Shows vs What's Actually On-Chain

| What MetaMask Shows | What's Stored On-Chain |
|---|---|
| "Sent to 0x7c5b..." | `to: 0x7c5b...` |
| "10 ETH" | `value: 0x8AC7230489E80000` (hex Wei) |
| "Gas fee: 0.000021 ETH" | `gas: 21000`, `gasPrice: 1000000000` |
| "Confirmed" | `status: 0x1` in receipt |
| "Block 23" | `blockNumber: 0x17` |
| ENS name if resolved | Raw address in actual tx |

---

## 3.4 Testing Procedure — Step by Step

### Test 1: Send from MetaMask to Interface B

1. Open MetaMask (connected to XDC-T)
2. Open your Interface B web page and click "Connect" then "Watch for Incoming TX"
3. In MetaMask, send 1 ETH to User B's address
4. Observe:
   - MetaMask shows "Pending" then "Confirmed"
   - Interface B detects the incoming transaction within 5–10 seconds
   - Interface B's balance updates

### Test 2: Read the Raw Transaction Data

While watching Interface B, also run this in your terminal after the transaction:

```bash
# Replace TX_HASH with the actual hash from MetaMask
curl -X POST http://127.0.0.1:8545 \
  -H "Content-Type: application/json" \
  --data '{
    "jsonrpc":"2.0",
    "method":"eth_getTransactionReceipt",
    "params":["0xTX_HASH_HERE"],
    "id":1
  }'
```

Document what each field means and what it reveals.

### Test 3: Block Explorer Simulation

Write a simple script that simulates what a block explorer does — read every transaction in a block and display it:

```javascript
// save as block_explorer.js
const { ethers } = require('ethers');

const provider = new ethers.JsonRpcProvider('http://127.0.0.1:8545');

async function exploreBlock(blockNumber) {
    const block = await provider.getBlock(blockNumber, true);
    console.log(`\n=== Block ${block.number} ===`);
    console.log(`Timestamp: ${new Date(block.timestamp * 1000).toISOString()}`);
    console.log(`Transactions: ${block.transactions.length}`);
    console.log(`Gas Used: ${block.gasUsed}`);
    console.log(`Miner: ${block.miner}`);

    for (const tx of block.transactions) {
        console.log(`\n  TX: ${tx.hash}`);
        console.log(`  From: ${tx.from}`);
        console.log(`  To:   ${tx.to}`);
        console.log(`  ETH:  ${ethers.formatEther(tx.value)}`);
    }
}

async function main() {
    const latest = await provider.getBlockNumber();
    for (let i = Math.max(0, latest - 5); i <= latest; i++) {
        await exploreBlock(i);
    }
}

main().catch(console.error);
```

```bash
npm install ethers
node block_explorer.js
```

---

# 4. Bot Integration & Testing

## 4.1 What the Bot Does

We will build a bot (automated client) that:
1. Connects to your Geth node via RPC
2. Resolves names from the NameRegistry contract
3. Initiates transactions to resolved addresses
4. Tracks transaction status until confirmation
5. Reports results

Think of it like an automated payment agent — given a name and amount, it handles everything else.

---

## 4.2 Architecture

```
┌─────────────────────────────────────────────────────┐
│                     XDC Bot                          │
│                                                      │
│  Input: "pay alice.xdc 5 ETH"                        │
│                                                      │
│  1. NameResolver        → calls NameRegistry contract│
│     "alice" → 0x9Be2...                              │
│                                                      │
│  2. TransactionBuilder  → builds raw tx              │
│     {from, to, value, gas, nonce}                    │
│                                                      │
│  3. TransactionSender   → submits via eth_sendTx     │
│     returns txHash                                   │
│                                                      │
│  4. StatusTracker       → polls for receipt          │
│     pending → mined → confirmed                      │
│                                                      │
│  Output: "Sent 5 ETH to alice.xdc (0x9Be2...)"       │
│          "TX: 0xabc... | Block 42 | Gas used: 21000" │
└─────────────────────────────────────────────────────┘
```

---

## 4.3 Building the Bot

Create a folder `xdc-bot/` and set it up:

```bash
mkdir xdc-bot
cd xdc-bot
npm init -y
npm install ethers readline-sync
```

Create `bot.js`:

```javascript
/**
 * XDC Bot — Automated payment agent for your private chain
 *
 * Features:
 * - Name resolution via NameRegistry contract
 * - Send ETH by name or raw address
 * - Track transaction until confirmed
 * - Show full transaction details
 */

const { ethers } = require('ethers');
const readline = require('readline');
const fs = require('fs');

// ── Configuration ──────────────────────────────────────
const CONFIG = {
    rpc: 'http://127.0.0.1:8545',
    fromAddress: '0xYOUR_ADDRESS_HERE',  // Bot's sending account
    contractAddress: fs.existsSync('./contract_address.txt')
        ? fs.readFileSync('./contract_address.txt', 'utf8').trim()
        : null,
    // Polling interval for tx confirmation (ms)
    pollInterval: 2000,
    // Max polls before timeout
    maxPolls: 30,
};

// ── Name Registry ABI (only functions we use) ──────────
const REGISTRY_ABI = [
    "function resolve(string name) view returns (address)",
    "function isAvailable(string name) view returns (bool)",
    "function register(string name, address resolved)",
    "function reverseLookup(address addr) view returns (string)",
    "function getRecord(string name) view returns (address owner, address resolved, uint256 registeredAt)"
];

// ── Setup ───────────────────────────────────────────────
let provider;
let registryContract;

async function setup() {
    provider = new ethers.JsonRpcProvider(CONFIG.rpc);

    // Test connection
    const blockNumber = await provider.getBlockNumber();
    const network = await provider.getNetwork();
    console.log(`Connected to node`);
    console.log(`  Chain ID: ${network.chainId}`);
    console.log(`  Block: ${blockNumber}`);

    // Set up registry contract if deployed
    if (CONFIG.contractAddress) {
        registryContract = new ethers.Contract(
            CONFIG.contractAddress,
            REGISTRY_ABI,
            provider
        );
        console.log(`  Registry: ${CONFIG.contractAddress}`);
    } else {
        console.log('  Registry: Not deployed (name resolution unavailable)');
    }

    const balance = await provider.getBalance(CONFIG.fromAddress);
    console.log(`  Bot balance: ${ethers.formatEther(balance)} ETH`);
    console.log('');
}

// ── Name Resolution ─────────────────────────────────────
async function resolveName(name) {
    if (!registryContract) {
        throw new Error('NameRegistry contract not deployed');
    }

    // Strip .xdc suffix if present
    const cleanName = name.endsWith('.xdc') ? name.slice(0, -4) : name;

    const address = await registryContract.resolve(cleanName);

    if (address === ethers.ZeroAddress) {
        throw new Error(`Name "${cleanName}.xdc" is not registered`);
    }

    return { name: cleanName + '.xdc', address };
}

// ── Transaction Sending ─────────────────────────────────
async function sendPayment(toAddress, amountEth) {
    const value = ethers.parseEther(amountEth.toString());

    // Get current nonce
    const nonce = await provider.getTransactionCount(CONFIG.fromAddress);
    const feeData = await provider.getFeeData();

    console.log(`Building transaction:`);
    console.log(`  From: ${CONFIG.fromAddress}`);
    console.log(`  To:   ${toAddress}`);
    console.log(`  ETH:  ${amountEth}`);
    console.log(`  Nonce: ${nonce}`);
    console.log(`  Gas Price: ${ethers.formatUnits(feeData.gasPrice, 'gwei')} Gwei`);

    // Send via RPC
    // Note: We use eth_sendTransaction which requires the account to be unlocked in Geth
    const txHash = await provider.send('eth_sendTransaction', [{
        from: CONFIG.fromAddress,
        to: toAddress,
        value: '0x' + value.toString(16),
        gas: '0x5208',  // 21000 — standard ETH transfer
    }]);

    return txHash;
}

// ── Transaction Tracking ────────────────────────────────
async function trackTransaction(txHash) {
    console.log(`\nTracking: ${txHash}`);
    console.log('Status: PENDING...');

    for (let i = 0; i < CONFIG.maxPolls; i++) {
        await new Promise(r => setTimeout(r, CONFIG.pollInterval));

        const receipt = await provider.getTransactionReceipt(txHash);

        if (receipt) {
            const status = receipt.status === 1 ? 'SUCCESS ✓' : 'FAILED ✗';
            console.log(`Status: ${status}`);
            console.log(`  Block: ${receipt.blockNumber}`);
            console.log(`  Gas Used: ${receipt.gasUsed.toString()}`);
            console.log(`  TX Hash: ${receipt.hash}`);

            // Try reverse lookup on recipient
            if (registryContract) {
                const recipientName = await registryContract.reverseLookup(receipt.to);
                if (recipientName) {
                    console.log(`  Recipient name: ${recipientName}.xdc`);
                }
            }

            return receipt;
        }

        process.stdout.write('.');
    }

    throw new Error('Transaction not mined within timeout');
}

// ── Bot Commands ────────────────────────────────────────
const COMMANDS = {
    help: {
        description: 'Show available commands',
        usage: 'help',
        run: async () => {
            console.log('\nAvailable commands:');
            for (const [cmd, info] of Object.entries(COMMANDS)) {
                console.log(`  ${cmd.padEnd(15)} ${info.description}`);
                console.log(`  ${''.padEnd(15)} Usage: ${info.usage}`);
            }
        }
    },

    balance: {
        description: 'Check balance of an address or name',
        usage: 'balance <address or name>',
        run: async (args) => {
            let address = args[0];
            let displayName = address;

            if (!address.startsWith('0x')) {
                const resolved = await resolveName(address);
                address = resolved.address;
                displayName = `${resolved.name} (${address})`;
            }

            const balance = await provider.getBalance(address);
            const nonce = await provider.getTransactionCount(address);
            console.log(`Balance of ${displayName}:`);
            console.log(`  ETH: ${ethers.formatEther(balance)}`);
            console.log(`  Nonce: ${nonce}`);
        }
    },

    pay: {
        description: 'Send ETH to a name or address',
        usage: 'pay <name or address> <amount in ETH>',
        run: async (args) => {
            if (args.length < 2) throw new Error('Usage: pay <name/address> <amount>');

            let toAddress = args[0];
            const amount = args[1];
            let displayName = toAddress;

            // Resolve name if not a raw address
            if (!toAddress.startsWith('0x')) {
                console.log(`Resolving "${toAddress}"...`);
                const resolved = await resolveName(toAddress);
                toAddress = resolved.address;
                displayName = resolved.name;
                console.log(`Resolved to: ${toAddress}`);
            }

            const txHash = await sendPayment(toAddress, amount);
            console.log(`\nTransaction submitted: ${txHash}`);
            const receipt = await trackTransaction(txHash);

            console.log(`\nPayment complete!`);
            console.log(`Sent ${amount} ETH to ${displayName}`);
        }
    },

    lookup: {
        description: 'Look up a name in the registry',
        usage: 'lookup <name>',
        run: async (args) => {
            if (!args[0]) throw new Error('Usage: lookup <name>');
            const name = args[0];

            if (!registryContract) {
                throw new Error('Registry not deployed');
            }

            const cleanName = name.endsWith('.xdc') ? name.slice(0, -4) : name;
            const available = await registryContract.isAvailable(cleanName);

            if (available) {
                console.log(`"${cleanName}.xdc" is available for registration`);
            } else {
                const [owner, resolved, registeredAt] = await registryContract.getRecord(cleanName);
                console.log(`Name: ${cleanName}.xdc`);
                console.log(`  Owner: ${owner}`);
                console.log(`  Resolves to: ${resolved}`);
                console.log(`  Registered at block: ${registeredAt.toString()}`);
            }
        }
    },

    reverse: {
        description: 'Reverse lookup: find name for an address',
        usage: 'reverse <address>',
        run: async (args) => {
            if (!args[0]) throw new Error('Usage: reverse <address>');
            if (!registryContract) throw new Error('Registry not deployed');

            const name = await registryContract.reverseLookup(args[0]);
            if (name) {
                console.log(`${args[0]} → ${name}.xdc`);
            } else {
                console.log(`No primary name set for ${args[0]}`);
            }
        }
    },

    tx: {
        description: 'Get details of a transaction',
        usage: 'tx <hash>',
        run: async (args) => {
            if (!args[0]) throw new Error('Usage: tx <hash>');

            const tx = await provider.getTransaction(args[0]);
            if (!tx) { console.log('Transaction not found'); return; }

            const receipt = await provider.getTransactionReceipt(args[0]);

            console.log(`Transaction: ${args[0]}`);
            console.log(`  From: ${tx.from}`);
            console.log(`  To: ${tx.to}`);
            console.log(`  Value: ${ethers.formatEther(tx.value)} ETH`);
            console.log(`  Gas Limit: ${tx.gasLimit.toString()}`);
            console.log(`  Gas Price: ${ethers.formatUnits(tx.gasPrice, 'gwei')} Gwei`);
            console.log(`  Nonce: ${tx.nonce}`);

            if (receipt) {
                console.log(`  Status: ${receipt.status === 1 ? 'SUCCESS' : 'FAILED'}`);
                console.log(`  Block: ${receipt.blockNumber}`);
                console.log(`  Gas Used: ${receipt.gasUsed.toString()}`);
            } else {
                console.log(`  Status: PENDING`);
            }
        }
    },

    blocks: {
        description: 'Show recent blocks',
        usage: 'blocks [count]',
        run: async (args) => {
            const count = parseInt(args[0] || '5');
            const latest = await provider.getBlockNumber();

            for (let i = latest; i > Math.max(0, latest - count); i--) {
                const block = await provider.getBlock(i);
                console.log(`Block ${block.number}: ${block.transactions.length} tx | ${new Date(block.timestamp * 1000).toLocaleTimeString()}`);
            }
        }
    }
};

// ── REPL (Interactive Shell) ─────────────────────────────
async function startREPL() {
    const rl = readline.createInterface({
        input: process.stdin,
        output: process.stdout
    });

    console.log('XDC Bot ready. Type "help" for commands.');
    console.log('');

    const prompt = () => {
        rl.question('xdc> ', async (input) => {
            const parts = input.trim().split(/\s+/);
            const cmd = parts[0].toLowerCase();
            const args = parts.slice(1);

            if (cmd === 'exit' || cmd === 'quit') {
                console.log('Goodbye.');
                rl.close();
                process.exit(0);
            }

            if (COMMANDS[cmd]) {
                try {
                    await COMMANDS[cmd].run(args);
                } catch (err) {
                    console.error(`Error: ${err.message}`);
                }
            } else if (cmd) {
                console.log(`Unknown command: ${cmd}. Type "help" for commands.`);
            }

            console.log('');
            prompt();
        });
    };

    prompt();
}

// ── Entry Point ──────────────────────────────────────────
async function main() {
    console.log('=== XDC Bot ===\n');
    await setup();
    await startREPL();
}

main().catch(err => {
    console.error('Fatal error:', err.message);
    process.exit(1);
});
```

Run the bot:
```bash
node bot.js
```

### Example bot session:

```
=== XDC Bot ===

Connected to node
  Chain ID: 12345
  Block: 174
  Registry: 0xCONTRACT_ADDRESS
  Bot balance: 999999.0 ETH

XDC Bot ready. Type "help" for commands.

xdc> help
  help           Show available commands
  balance        Check balance of an address or name
  pay            Send ETH to a name or address
  lookup         Look up a name in the registry
  reverse        Reverse lookup: find name for an address
  tx             Get details of a transaction
  blocks         Show recent blocks

xdc> lookup alice
Name: alice.xdc
  Owner: 0x9Be2...
  Resolves to: 0x9Be2...
  Registered at block: 45

xdc> pay alice.xdc 5
Resolving "alice.xdc"...
Resolved to: 0x9Be2...
Building transaction:
  From: 0x7c5b...
  To:   0x9Be2...
  ETH:  5
  Nonce: 2
  Gas Price: 1 Gwei

Transaction submitted: 0xabc123...
Tracking: 0xabc123...
Status: PENDING......
Status: SUCCESS ✓
  Block: 178
  Gas Used: 21000
  TX Hash: 0xabc123...

Payment complete!
Sent 5 ETH to alice.xdc
```

---

# 5. Multi-Client Node Testing

## 5.1 Why Multi-Client Testing Matters

A blockchain's strength comes partly from **client diversity**. If every node runs the same software, a bug in that software could take down the entire network simultaneously. Running multiple different clients means:
- A bug in Geth doesn't affect Nethermind nodes
- The network stays alive even if one client has an issue
- You can compare results across clients to detect consensus bugs

On Ethereum mainnet, the goal is to have no single client exceed ~33% of nodes — that way no single bug can break consensus.

---

## 5.2 The Clients We Will Test

### Geth (Go Ethereum)
- Language: Go
- Maintained by: Ethereum Foundation
- The client you've been using
- Market share on mainnet: ~40%

### Nethermind
- Language: C# / .NET
- Maintained by: Nethermind team
- Known for: high performance, good Windows support, rich plugin system
- Market share on mainnet: ~15%

### Reth (Rust Ethereum)
- Language: Rust
- Maintained by: Paradigm
- Known for: extremely fast sync, memory efficient, modular architecture
- Market share: growing, newer entrant
- Note: Reth v1.0 was released in 2024

---

## 5.3 Important Limitation: Clique PoA Compatibility

Before building a multi-client setup, you must understand a fundamental compatibility issue:

**Nethermind does support Clique PoA.** It has a Clique consensus implementation and can participate in your private network.

**Reth does NOT support Clique PoA.** Reth only supports PoS networks. For a multi-client setup including Reth, you would need to switch to a PoS devnet configuration using a consensus layer client.

| Client | Clique PoA | PoS (with CL) |
|---|---|---|
| Geth v1.13.15 | ✅ Yes | ✅ Yes |
| Nethermind | ✅ Yes | ✅ Yes |
| Reth | ❌ No | ✅ Yes |

---

## 5.4 Architecture: 3-Node Clique Network (2 Geth + 1 Nethermind)

```
┌─────────────────────────────────────────────────────────┐
│                  Private Network: XDC-T                  │
│                   Chain ID: 12345                        │
│                                                          │
│  Node 1 (Geth v1.13.15)          Node 2 (Geth v1.13.15) │
│  Port: 30303                     Port: 30304             │
│  RPC: 8545                       RPC: 8546               │
│  Role: Sealer (signer)           Role: Sealer (signer)   │
│              │                         │                  │
│              └─────────┬───────────────┘                  │
│                        │                                  │
│                        │ P2P Network (devp2p)             │
│                        │                                  │
│              ┌─────────┴──────────┐                       │
│              │  Node 3 (Nethermind)│                      │
│              │  Port: 30305       │                       │
│              │  RPC: 8547         │                       │
│              │  Role: Sync only   │                       │
│              └────────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

In Clique PoA, only designated signers can seal blocks. For a 3-node network with 2 Geth signers, you need both Geth nodes listed as signers in `extradata`. Nethermind will sync and validate but not seal blocks.

---

## 5.5 Setup: Multi-Signer Genesis File

For 2 Geth signers, the `extradata` field must include BOTH signer addresses:

```
extradata format:
[32 zero bytes][address1 (20 bytes)][address2 (20 bytes)][65 zero bytes]
```

In hex:
```
0x
0000000000000000000000000000000000000000000000000000000000000000
ADDRESS1_WITHOUT_0x
ADDRESS2_WITHOUT_0x
0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

### Step 1 — Create Accounts for Both Geth Nodes

```bash
# Create accounts for Node 1
mkdir node1 node2 node3
geth --datadir node1 account new
# Note the address: e.g. 0xAAAA...

geth --datadir node2 account new
# Note the address: e.g. 0xBBBB...
```

### Step 2 — Create Multi-Signer genesis.json

```json
{
  "config": {
    "chainId": 12345,
    "homesteadBlock": 0,
    "eip150Block": 0,
    "eip155Block": 0,
    "eip158Block": 0,
    "byzantiumBlock": 0,
    "constantinopleBlock": 0,
    "petersburgBlock": 0,
    "istanbulBlock": 0,
    "berlinBlock": 0,
    "londonBlock": 0,
    "clique": { "period": 5, "epoch": 30000 }
  },
  "difficulty": "1",
  "gasLimit": "8000000",
  "extradata": "0x0000000000000000000000000000000000000000000000000000000000000000AAAA_ADDRESS_NO_0xBBBB_ADDRESS_NO_0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
  "alloc": {
    "0xAAAA_FULL_ADDRESS": { "balance": "500000000000000000000000000" },
    "0xBBBB_FULL_ADDRESS": { "balance": "500000000000000000000000000" }
  }
}
```

### Step 3 — Initialize All Nodes

```bash
# Initialize all three node directories with the same genesis
geth init --datadir node1 genesis.json
geth init --datadir node2 genesis.json

# Nethermind uses a different command (see below)
```

### Step 4 — Start Node 1 (Geth — Signer 1)

```bash
geth \
  --datadir node1 \
  --networkid 12345 \
  --port 30303 \
  --http --http.port 8545 \
  --http.api admin,eth,web3,net,personal,miner \
  --nodiscover \
  --unlock 0xAAAA_ADDRESS \
  --allow-insecure-unlock \
  --miner.etherbase 0xAAAA_ADDRESS \
  --mine \
  --verbosity 3 \
  2>node1.log &

# Get Node 1's enode address (needed to connect other nodes)
geth attach node1/geth.ipc --exec "admin.nodeInfo.enode"
# Output: "enode://PUBKEY@127.0.0.1:30303"
```

### Step 5 — Start Node 2 (Geth — Signer 2)

```bash
# Replace ENODE_FROM_NODE1 with the enode from Step 4
geth \
  --datadir node2 \
  --networkid 12345 \
  --port 30304 \
  --http --http.port 8546 \
  --http.api admin,eth,web3,net,personal,miner \
  --nodiscover \
  --bootnodes "ENODE_FROM_NODE1" \
  --unlock 0xBBBB_ADDRESS \
  --allow-insecure-unlock \
  --miner.etherbase 0xBBBB_ADDRESS \
  --mine \
  --verbosity 3 \
  2>node2.log &
```

### Step 6 — Install and Start Nethermind

**Download Nethermind:**
```bash
# Linux
wget https://github.com/NethermindEth/nethermind/releases/latest/download/nethermind-linux-x64.zip
unzip nethermind-linux-x64.zip -d nethermind/

# Windows
# Download from: https://github.com/NethermindEth/nethermind/releases
```

**Create Nethermind config file** `node3/nethermind_xdc.json`:

```json
{
  "Init": {
    "ChainSpecPath": "./chainspec.json",
    "BaseDbPath": "./node3/db",
    "LogFileName": "node3.log"
  },
  "Network": {
    "DiscoveryPort": 30305,
    "P2PPort": 30305,
    "StaticPeers": "ENODE_FROM_NODE1,ENODE_FROM_NODE2"
  },
  "JsonRpc": {
    "Enabled": true,
    "Port": 8547,
    "Host": "127.0.0.1",
    "EnabledModules": ["Eth", "Net", "Web3", "Admin"]
  }
}
```

**Create Nethermind chain spec** `chainspec.json` (Nethermind uses a different genesis format called OpenEthereum/Parity chain spec):

```json
{
  "name": "XDC-T",
  "engine": {
    "clique": {
      "params": {
        "period": 5,
        "epoch": 30000
      }
    }
  },
  "params": {
    "gasLimitBoundDivisor": "0x400",
    "maxCodeSize": "0x6000",
    "chainID": "0x3039"
  },
  "genesis": {
    "seal": {
      "ethereum": {
        "nonce": "0x0000000000000000",
        "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000"
      }
    },
    "difficulty": "0x1",
    "author": "0x0000000000000000000000000000000000000000",
    "timestamp": "0x0",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "extraData": "SAME_EXTRADATA_AS_GENESIS_JSON",
    "gasLimit": "0x7A1200"
  },
  "accounts": {
    "0xAAAA_ADDRESS": { "balance": "500000000000000000000000000" },
    "0xBBBB_ADDRESS": { "balance": "500000000000000000000000000" }
  }
}
```

**Start Nethermind:**
```bash
./nethermind/nethermind \
  --config node3/nethermind_xdc.json
```

---

## 5.6 Verifying Synchronization

Once all three nodes are running, verify they are in sync:

```bash
# Check block numbers across all nodes
curl -s -X POST http://127.0.0.1:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | python3 -m json.tool

curl -s -X POST http://127.0.0.1:8546 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | python3 -m json.tool

curl -s -X POST http://127.0.0.1:8547 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' | python3 -m json.tool
```

All three should return the same block number (within 1–2 blocks of each other).

### Check Peer Count

```bash
# Node 1 should have 2 peers (Node 2 and Node 3)
curl -s -X POST http://127.0.0.1:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'
# Expected: {"result":"0x2"}
```

---

## 5.7 Testing Interoperability

### Test A: Send from Geth Node, Verify on Nethermind

```bash
# Send transaction from Node 1 (Geth)
TX_HASH=$(curl -s -X POST http://127.0.0.1:8545 \
  -H "Content-Type: application/json" \
  --data '{
    "jsonrpc":"2.0","method":"eth_sendTransaction",
    "params":[{"from":"0xAAAA_ADDRESS","to":"0xBBBB_ADDRESS","value":"0xDE0B6B3A7640000","gas":"0x5208"}],
    "id":1
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['result'])")

echo "TX Hash: $TX_HASH"

# Wait 6 seconds for mining
sleep 6

# Verify receipt on Nethermind (Node 3)
curl -s -X POST http://127.0.0.1:8547 \
  -H "Content-Type: application/json" \
  --data "{\"jsonrpc\":\"2.0\",\"method\":\"eth_getTransactionReceipt\",\"params\":[\"$TX_HASH\"],\"id\":1}" | python3 -m json.tool
```

If Nethermind shows the same receipt that Geth shows, the two clients are in sync and agree on the chain state.

### Test B: Compare Block Hashes

Both clients must have identical block hashes for the same block number. Any difference means a consensus fork:

```bash
# Get block 10 hash from Geth
GETH_HASH=$(curl -s -X POST http://127.0.0.1:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["0xa",false],"id":1}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['result']['hash'])")

# Get block 10 hash from Nethermind
NETH_HASH=$(curl -s -X POST http://127.0.0.1:8547 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["0xa",false],"id":1}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin)['result']['hash'])")

echo "Geth:       $GETH_HASH"
echo "Nethermind: $NETH_HASH"

if [ "$GETH_HASH" = "$NETH_HASH" ]; then
  echo "MATCH ✓ — Clients agree on chain state"
else
  echo "MISMATCH ✗ — Clients have forked!"
fi
```

---

## 5.8 Performance and Observations to Document

When running your multi-client setup, record these observations:

| Metric | Geth Node 1 | Geth Node 2 | Nethermind |
|---|---|---|---|
| Block sync lag (vs latest) | | | |
| Memory usage (MB) | | | |
| CPU usage (%) | | | |
| Peer connections | | | |
| RPC response time (ms) | | | |
| Startup time (seconds) | | | |

**How to measure memory and CPU:**
```bash
# Linux
ps aux | grep geth
ps aux | grep nethermind

# Or use top
top -p $(pgrep geth)
```

---

## 5.9 Known Compatibility Issues to Watch For

**Issue 1: extradata format differences**  
Geth and Nethermind may interpret extradata slightly differently. If Nethermind refuses to sync, check that the chainspec extradata exactly matches the genesis.json extradata.

**Issue 2: API naming differences**  
Some methods have slightly different names or return formats between clients:

| Method | Geth Behaviour | Nethermind Behaviour |
|---|---|---|
| `eth_accounts` | Returns local node accounts | May return empty array |
| `eth_mining` | Returns true/false | May always return false |
| `net_version` | Returns chain ID as string | Same |

**Issue 3: Clique epoch handling**  
At each epoch boundary (every 30,000 blocks by default), signers are recalculated. Both Geth and Nethermind should handle this identically, but it is worth monitoring.

---

# Appendix — Quick Reference

## Hex Conversion

```
0x1      = 1
0xa      = 10
0x10     = 16
0x3039   = 12345
0x5208   = 21000  (gas for ETH transfer)
0xDE0B6B3A7640000 = 1 ETH in Wei
0x8AC7230489E80000 = 10 ETH in Wei
```

## Key Addresses & Values (Your Network)

```
Chain ID:    12345
RPC:         http://127.0.0.1:8545
IPC (Win):   \\.\pipe\geth.ipc
IPC (Linux): XDC-T/geth.ipc
Block time:  5 seconds (Clique period)
Gas limit:   8,000,000 per block
```

## File Structure Reference

```
XDC-T/
├── geth/
│   ├── chaindata/        ← Block data
│   ├── lightchaindata/
│   └── nodes/            ← Peer data
├── keystore/             ← ⚠️ Back this up — your private keys
│   └── UTC--2024-...     ← Encrypted keystore file
└── geth.ipc              ← IPC socket (created when node runs)

Project files:
├── genesis.json          ← Chain configuration
├── NameRegistry.sol      ← Smart contract source
├── compiled/             ← ABI + bytecode after solcjs
├── contract_address.txt  ← Deployed contract address
├── deploy.js             ← Deployment script
├── xdc-bot/
│   ├── bot.js            ← Interactive bot
│   └── package.json
└── interface-b/
    └── index.html        ← User B web interface
```

## Geth Version Decision Tree

```
Do you need Clique PoA?
├── Yes → Use Geth v1.13.15
│         (last version with Clique)
└── No  → Do you want PoS?
           ├── Yes → Geth v1.14+ with consensus client
           └── No  → Consider alternatives (Hardhat, Foundry for dev)
```

## Common Error Quick Fix

| Error | Fix |
|---|---|
| `integer divide by zero` | extradata missing signer address |
| `Geth only supports PoS` | Using Geth v1.14+, downgrade to v1.13.15 |
| `authentication needed` | Add `--allow-insecure-unlock` |
| `etherbase must be specified` | Add `--miner.etherbase 0xADDR` |
| `datadir used by another process` | `pkill geth` or `Stop-Process -Name geth -Force` |
| `personal is not defined` | Use IPC instead of HTTP, or add `--allow-insecure-unlock` |
