# XDC Subnet — Docker & Native Validator Deployment

This repository contains a complete, reproducible setup for an XDPoS private subnet (chainId `7670`) running on a single host. It supports both:

- **Docker-based deployment** via `docker-compose` (original setup)
- **Native binary deployment** using a self-built `XDPoSChain` binary

---

## Repository Layout

```
~/subnet/
├── README.md                          # This file
├── start_xdpos.sh                     # Generator launcher: pulls the subnet-generator image and starts it
├── MIGRATION.md                       # Docker → native migration guide and implementation report
├── MIGRATION_PLAN.md                  # Original migration planning document
├── SUBNET_ANALYSIS.md                 # Deep analysis of the subnet configuration
├── MANUAL_DEPLOYMENT_PLAN.md          # Detailed manual deployment plan
├── generated/                         # Files produced by the subnet generator
│   ├── README.md                      # Generated documentation for the Docker subnet
│   ├── docker-compose.yml             # Docker Compose services (3 validators + bootnode + Redis)
│   ├── docker-up.sh                   # Start Docker subnet
│   ├── docker-down.sh                 # Stop Docker subnet
│   ├── gen.env                       # Basic network config (name, subnet count, IP)
│   ├── genesis_input.yml             # Human-readable genesis input
│   ├── genesis.json                   # Machine-readable genesis block (committed)
│   ├── bootnodes.list                 # Bootnode enode URL used by validators
│   ├── scripts/                       # Health-check scripts
│   │   ├── check-mining.sh
│   │   └── check-peer.sh
│   ├── start-native-validator1.sh     # Native validator 1 startup script
│   ├── start-native-validator2.sh     # Native validator 2 startup script
│   └── start-native-validator3.sh     # Native validator 3 startup script
└── manual/                            # Alternative manual deployment artifacts
    ├── IMPLEMENTATION_REPORT.md       # Report on the completed manual migration
    ├── scripts/                       # Manual migration and lifecycle scripts
    └── bootnodes/                     # Manual bootnode configuration
```

---

## Quick Start

### 1. Generate a New Subnet (Docker-first)

```bash
cd ~/subnet
./start_xdpos.sh
```

This pulls `xinfinorg/subnet-generator:v2.1.0` and starts a local web UI on port `5210`. Open `http://localhost:5210/gen_xdpos` to generate the Docker subnet files.

> ⚠️ This will create files under `generated/`. Runtime data such as `xdcchain*/`, `keys.json`, `masternode*.env`, and `.pwd` will be ignored by `.gitignore`.

### 2. Start the Docker Subnet

```bash
cd ~/subnet/generated
./docker-up.sh machine1
```

Verify:

```bash
./scripts/check-mining.sh
./scripts/check-peer.sh
```

### 3. Migrate to Native Binaries (Optional)

See [`MIGRATION.md`](MIGRATION.md) for the full step-by-step guide. In short:

1. Build the correct `XDPoSChain` commit (`53e56018250499fecb277f0049471f7990c50591` for `v2.6.1-beta`).
2. Stop the Docker validator containers.
3. Run the native startup scripts:

```bash
bash ~/subnet/generated/start-native-validator1.sh
bash ~/subnet/generated/start-native-validator2.sh
bash ~/subnet/generated/start-native-validator3.sh
```

4. Verify block production and peer counts as described in [`MIGRATION.md`](MIGRATION.md).

---

## Network Parameters

| Parameter | Value |
|-----------|-------|
| Network name | `xdcSubnet` |
| Chain ID | `7670` |
| Consensus | XDPoS v2 |
| Block time | 2 seconds |
| Epoch | 900 blocks |
| Masternodes | 3 |
| P2P ports | `20303`, `20304`, `20305` |
| RPC ports | `8545`, `8546`, `8547` |
| WebSocket ports | `9555`, `9556`, `9557` |

---

## Important Security Notes

- **Never commit `keys.json`, `masternode*.env`, or `.pwd`**. These files contain private keys and are ignored by `.gitignore`.
- The genesis file `generated/genesis.json` is committed because it is public chain configuration and required for reproducibility.
- Validator data directories (`generated/xdcchain*/`) are large and runtime-specific; they are ignored.

---

## Useful Commands

| Task | Command |
|------|---------|
| Start Docker subnet | `./generated/docker-up.sh machine1` |
| Stop Docker subnet | `./generated/docker-down.sh machine1` |
| Start native validator 1 | `bash ~/subnet/generated/start-native-validator1.sh` |
| Check block height | `curl -s -X POST http://localhost:8545 -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'` |
| Check mining | `curl -s -X POST http://localhost:8545 -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"eth_mining","params":[],"id":1}'` |
| Check peers | `curl -s -X POST http://localhost:8545 -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}'` |

---

## Documentation Index

- [`generated/README.md`](generated/README.md) — Detailed reference for the generated Docker subnet.
- [`MIGRATION.md`](MIGRATION.md) — How the Docker validators were migrated to native binaries.
- [`MIGRATION_PLAN.md`](MIGRATION_PLAN.md) — Planning document for the migration.
- [`SUBNET_ANALYSIS.md`](SUBNET_ANALYSIS.md) — Analysis of the subnet configuration and version choices.
- [`MANUAL_DEPLOYMENT_PLAN.md`](MANUAL_DEPLOYMENT_PLAN.md) — Full manual deployment plan.
- [`manual/IMPLEMENTATION_REPORT.md`](manual/IMPLEMENTATION_REPORT.md) — Report on the manual migration outcome.
- [`LEGACY_CUSTOM_CHAIN_COMPATIBILITY.md`](LEGACY_CUSTOM_CHAIN_COMPATIBILITY.md) — v2.6.1→v2.8.2 compatibility root cause, fix, and validation for legacy custom subnets.

---

*Repository: `https://github.com/shhalaka/xdc-subnet`*
