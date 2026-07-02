# Sovereign L1 — Phase 1: Chain Scaffold & Genesis Configuration

> **Branch:** `phase-1`
> **Status:** 21 / 23 tasks complete · 2 scaffold stubs (deferred to Phase 2) · 0 missing

```bash
git clone --single-branch --branch phase-1 --depth 1 https://github.com/majednitol/Sovereign.git
```

---

## What Is Sovereign L1?

Sovereign L1 is a custom Cosmos SDK–based Layer-1 blockchain with a built-in EVM execution layer. It is dual-stack: every address works as both a native Cosmos `sov1…` address and a `0x…` EVM address. It runs CosmWasm smart contracts alongside Solidity contracts, uses IBC for cross-chain communication, and enforces a fixed-supply equal-power validator model.

---

## What Phase 1 Covers

Phase 1 is the **chain scaffold and genesis configuration** — the foundational wiring that must be correct before any application logic is layered on. No Phase 2 features (bridge, oracle, certification, settlement, governance-ext, milestone) are functional in this branch; they exist as `.gitkeep` stubs only.

**Phase 1 = "Can the chain start with the right parameters and not panic?"**

| Area | What was done |
|------|---------------|
| **Dependency pinning** | `cosmos/evm@v0.7.0` pinned; `skip-mev/feemarket` removed; `ibc-go` upgraded to v11.1.0 |
| **Keeper wiring** | `x/feemarket`, `x/vm`, `x/erc20`, `ibcKeeper`, `ibcTransferKeeper`, `ibcFeeKeeper` all wired into `app.go` with store keys, module registration, and correct BeginBlocker / EndBlocker / InitGenesis order |
| **Ante handler** | `evmante.NewAnteHandler` with full `HandlerOptions`; authz blocked-message map includes `MsgEthereumTx` and 5 native bridge/oracle/settlement message types |
| **Bech32 prefixes** | Explicitly set to `sov`, `sovpub`, `sovvaloper`, `sovvaloperpub`, `sovvalcons`, `sovvalconspub` — not SDK defaults |
| **EVM parameters** | Chain ID `7777`, denom `atoken`, `EnableCreate = true`, `AllowUnprotectedTxs = false` |
| **Fee market parameters** | `NoBaseFee = false`, `ElasticityMultiplier = 2`, `EnableHeight = 0` |
| **CosmWasm** | `CodeUploadAccess = Nobody` in genesis (no permissionless uploads) |
| **Equal-power validators** | Every active validator gets fixed voting power of `1,000,000`; `AllocateTokens` splits rewards equally; `HistoricalInfo` overridden to match |
| **Upgrade handler** | `v1.0.0` upgrade handler with a real `module.Configurator` (not a nil stub) |
| **Genesis script** | `scripts/generate_genesis.go` — 453-line script with `--verify`, `--out`, `--chain-id` flags; runs INV-1 through INV-5 supply invariants |
| **JSON-RPC** | Enabled on ports `8545` (HTTP) / `8546` (WS) via `app.toml` |
| **Documentation** | 13 ADRs, genesis parameter reference, ops runbooks, security threat model, testnet onboarding, mainnet chain-registry |

---

## File & Folder Structure

```
Sovereign/                          ← repo root
│
├── chain/                          ← Cosmos SDK chain daemon (Go module: github.com/sovereign-l1/chain)
│   ├── app/
│   │   ├── app.go                  ← App struct, keeper wiring, module manager, ante handler
│   │   ├── abci.go                 ← ABCI hook overrides
│   │   ├── export.go               ← State export for genesis snapshots
│   │   ├── upgrades.go             ← v1.0.0 upgrade handler (uses app.Configurator)
│   │   ├── staking_compatibility.go← Equal-power validator logic (GetEqualizedValidatorPower,
│   │   │                             AllocateTokens, OverrideHistoricalInfo)
│   │   ├── wasm.go                 ← CosmWasm config (CodeUploadAccess, ConstitutionContractAddr)
│   │   ├── app_test.go
│   │   ├── simulation_test.go
│   │   └── wasm_integration_test.go
│   │
│   ├── cmd/chaind/
│   │   └── main.go                 ← Entry point; setupSDKConfig() sets Bech32 prefixes ("sov…")
│   │
│   ├── config/
│   │   └── app.toml                ← JSON-RPC ports 8545/8546, gas-cap, namespaces, mempool config
│   │
│   ├── ibcfee/                     ← Local ICS-29 stub (Phase 1 scaffold; pass-through middleware)
│   │   ├── fee.go                  ← AppModule + NewIBCMiddleware (returns app unchanged)
│   │   ├── keeper/keeper.go        ← Empty Keeper struct
│   │   ├── types/types.go          ← Minimal type stubs
│   │   └── go.mod                  ← Local module (chain/ibcfee)
│   │
│   ├── x/                          ← Custom Cosmos modules
│   │   ├── erc20/types.go          ← Token pair type definitions
│   │   ├── feemarket/types.go      ← Fee market type definitions
│   │   ├── vm/precompiles/         ← EVM precompile placeholder (.gitkeep)
│   │   ├── bridge/                 ← Phase 2 stub (.gitkeep)
│   │   ├── certification/          ← Phase 2 stub (.gitkeep)
│   │   ├── governance-ext/         ← Phase 2 stub (.gitkeep)
│   │   ├── milestone/              ← Phase 2 stub (.gitkeep)
│   │   ├── oracle/                 ← Phase 2 stub (.gitkeep)
│   │   ├── settlement/             ← Phase 2 stub (.gitkeep)
│   │   └── validator/              ← Phase 2 stub (.gitkeep)
│   │
│   ├── api/                        ← gRPC / REST protobuf generated code
│   │   ├── backend/v1/             ← Backend query/stream proto stubs
│   │   ├── explorer/v1/            ← Explorer query proto stubs
│   │   └── relayer/v1/             ← Relayer query proto stubs
│   │
│   ├── genesis.json                ← Committed Phase 1 genesis (generated by generate_genesis.go)
│   ├── Dockerfile                  ← Builds chaind binary
│   ├── entrypoint.sh               ← Devnet init script (keys, gentx, collect-gentxs, app.toml patches)
│   ├── go.mod                      ← Go module (pinned versions; local replace for chain/ibcfee)
│   └── go.sum
│
├── scripts/
│   └── generate_genesis.go         ← Genesis generation + invariant verification (INV-1 to INV-5)
│
├── e2e/                            ← End-to-end test suite (separate Go module)
│   ├── phase_0_verification_test.go
│   ├── phase_1_verification_test.go
│   ├── phase_1_integration_test.go
│   ├── explorer_phase_1_test.go
│   ├── explorer_phase_1_integration_test.go
│   ├── go.mod
│   └── go.sum
│
├── artifacts/                      ← Compiled CosmWasm contract binaries (.wasm)
│   ├── constitution.wasm
│   ├── governance.wasm
│   ├── reserve_fund.wasm
│   ├── treasury.wasm
│   ├── cw_counter.wasm
│   └── checksums.txt
│
├── doc/                            ← All project documentation
│   ├── adr/                        ← Architecture Decision Records (ADR-001 to ADR-013)
│   │   ├── adr-001-validator-cardinality.md
│   │   ├── adr-002-certification-liveness.md
│   │   ├── adr-003-oracle-commit-reveal.md
│   │   ├── adr-004-bridge-security-model.md
│   │   ├── adr-005-cqrs-nats-topology.md
│   │   ├── adr-006-cosmwasm-governance.md
│   │   ├── adr-007-operational-security.md
│   │   ├── adr-008-version-pinning.md
│   │   ├── adr-009-evm-chain-id.md
│   │   ├── adr-010-fee-market-consolidation.md
│   │   ├── adr-011-evm-denomination.md
│   │   ├── adr-012-pagination-strategy.md
│   │   └── adr-013-grpc-gateway-deployment.md
│   ├── governance/
│   │   └── genesis_parameters.md   ← Supply, staking, Bech32, chain ID reference
│   ├── evm/
│   │   └── changelog_tracking.md
│   ├── ops/
│   │   ├── runbooks.md             ← Operational runbooks
│   │   ├── security_threat_model.md
│   │   └── audit_engagement.json
│   ├── testnet/
│   │   ├── onboarding.md
│   │   └── stability_checklist.md
│   ├── mainnet/
│   │   └── chain-registry.json     ← chainlist.org metadata
│   ├── phase_0_audit_report.md
│   └── phase_1_gap_analysis.md
│
├── docker-compose.yml              ← Full devnet stack (chain, NATS cluster, Postgres, explorer, frontend, Vault)
├── final-phase-1.md                ← Phase 1 agent instructions (committed to repo)
│
├── backend/                        ← Phase 2 stub (.gitkeep)
├── bridge/                         ← Phase 2 stub (.gitkeep)
├── contracts/                      ← Phase 2 Solidity contracts (.gitkeep)
├── db/                             ← Database schema (.gitkeep)
├── evm/                            ← Phase 2 EVM tooling (.gitkeep)
├── explorer/                       ← Phase 2 explorer (.gitkeep)
├── explorer-api/                   ← Phase 2 explorer API (.gitkeep)
├── explorer-indexer/               ← Phase 2 indexer (.gitkeep)
└── frontend/                       ← Phase 2 frontend (.gitkeep)
```

---

## Chain Parameters

| Parameter | Value |
|-----------|-------|
| **Cosmos Chain ID** | `sovereign-testnet-1` (testnet) · `sovereign-1` (mainnet) |
| **EVM Chain ID** | `7777` |
| **Native denom (Cosmos)** | `utoken` (6 decimals) |
| **Native denom (EVM gas)** | `atoken` (18 decimals) |
| **Total supply** | 1,000,000,000 TOKEN (fixed, 0% inflation) |
| **Account prefix** | `sov` |
| **Validator prefix** | `sovvaloper` |
| **BIP-44 derivation** | `m/44'/60'/0'/0/0` (Ethereum path) |
| **Active validator slots** | 30 (fixed cardinality) |
| **Validator voting power** | 1,000,000 each (equalized — ignores stake) |
| **Unbonding period** | 21 days |
| **EVM JSON-RPC (HTTP)** | Port `8545` |
| **EVM JSON-RPC (WS)** | Port `8546` |
| **CosmWasm code upload** | `Nobody` (governance-gated only) |
| **EIP-155 replay protection** | Enforced (`AllowUnprotectedTxs = false`) |

---

## Key Dependency Versions

| Package | Version |
|---------|---------|
| `cosmos/cosmos-sdk` | v0.54.3 |
| `cosmos/evm` | v0.7.0 |
| `cosmos/ibc-go/v11` | v11.1.0 |
| `CosmWasm/wasmd` | v0.70.2 |
| `cometbft/cometbft` | v0.39.3 |
| `ethereum/go-ethereum` | v1.17.0 |
| Go toolchain | 1.25.9 |

---

## How to Build & Run

### Quickstart — local node

**1. Start the local node**

```bash
./scripts/run_local_node.sh
```

**2. Verify block production is progressing** (open a second terminal)

```bash
curl -s http://localhost:26657/status | jq .result.sync_info.latest_block_height
```

The number should increment every ~2 seconds. If it stays at `0` or returns an error, the node is still initializing — wait a few seconds and retry.

**3. Verify EVM Chain ID is exactly 7777**

```bash
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  -H "Content-Type: application/json" http://localhost:8545
```

Expected response:

```json
{"jsonrpc":"2.0","id":1,"result":"0x1e61"}
```

`0x1e61` is `7777` in hex. Any other value means the genesis was not applied correctly — regenerate it with:

```bash
go run scripts/generate_genesis.go --chain-id sovereign-testnet-1 --out chain/genesis.json --verify
```

---

### Build the daemon binary

```bash
cd chain
go build -o chaind ./cmd/chaind
```

### Run a single-node devnet with Docker

```bash
docker build -t sovereign-chain ./chain
docker run -it \
  -e CHAIN_ID=sovereign-testnet-1 \
  -e MONIKER=local-node \
  -p 26657:26657 \
  -p 1317:1317 \
  -p 9090:9090 \
  -p 8545:8545 \
  -p 8546:8546 \
  sovereign-chain
```

`entrypoint.sh` handles first-run initialization automatically:
1. `chaind init` — creates chain home
2. Copies `genesis.json` if mounted at `/root/genesis.json`
3. Creates `validator` and `faucet` keys in the test keyring
4. Funds accounts and creates the genesis transaction (`gentx`)
5. Collects gentxs and patches `app.toml` / `config.toml` to listen on all interfaces

### Run the full devnet stack (chain + NATS + Postgres + Vault)

```bash
docker-compose up
```

Services spun up:
- `chain-node` — the Sovereign L1 daemon
- `nats-0/1/2` — 3-node NATS JetStream cluster (ports 4222, 8222)
- `db-write` — TimescaleDB write database (port 5433)
- `db-read` + `db-read-standby` + `pgbouncer-read` — read replica + connection pool (port 5434 / 6432)
- `vault` — HashiCorp Vault secret management (port 8200)
- `frontend` — Phase 2 frontend container (port 3000)

### Regenerate & verify genesis

```bash
go run scripts/generate_genesis.go \
  --chain-id sovereign-testnet-1 \
  --out chain/genesis.json \
  --verify
```

The `--verify` flag runs all 5 supply invariants:
- **INV-1** Total supply equals 1,000,000,000 TOKEN
- **INV-2** Cosmos allocation = Total supply − bridge escrow
- **INV-3** No permissionless CosmWasm uploads
- **INV-4** EVM Chain ID matches `7777`
- **INV-5** Fee market `no_base_fee = false`

### Run tests

```bash
# Unit tests
cd chain && go test ./...

# End-to-end tests
cd e2e && go test ./...

# Simulation (randomized)
cd chain && go test ./app -run TestAppSimulation -NumBlocks=500 -BlockSize=200 -Seed=$RANDOM
```

---

## CI Pipeline

`.github/workflows/ci.yml` runs on every push and pull request to `master`:

| Step | Tool |
|------|------|
| Go lint | `golangci-lint v1.60` |
| Protobuf lint | `buf lint` |
| Protobuf breaking-change check | `buf breaking` |
| Unit + E2E tests | `make test` |
| Binary build | `make build` |
| Simulation | `go test ./chain/app -run TestAppSimulation` |

---

## Architecture Decisions (ADRs)

| ADR | Decision |
|-----|----------|
| ADR-001 | Validator cardinality fixed at 30 (equal-power slots) |
| ADR-002 | Certification module liveness model |
| ADR-003 | Oracle commit-reveal scheme |
| ADR-004 | Bridge security model |
| ADR-005 | CQRS + NATS JetStream topology |
| ADR-006 | CosmWasm governance-gated contract deployment |
| ADR-007 | Operational security requirements |
| ADR-008 | Version pinning policy |
| ADR-009 | EVM Chain ID `7777` selected and registered |
| ADR-010 | Fee market consolidated to `cosmos/evm` (removed `skip-mev/feemarket`) |
| ADR-011 | EVM denomination set to `atoken` (18 decimals) |
| ADR-012 | Pagination strategy for gRPC/REST |
| ADR-013 | gRPC-gateway deployment topology |

Full text of all ADRs: [`doc/adr/`](doc/adr/)

---

## Phase Roadmap

| Phase | Branch | Focus | Status |
|-------|--------|-------|--------|
| **Phase 1** | `phase-1` | Chain scaffold, keeper wiring, genesis configuration | ✅ Complete (this branch) |
| **Phase 2** | — | Bridge, oracle, certification, settlement, governance-ext, milestone modules | 🔜 Next |
| **Phase 3** | — | Explorer, frontend, relayer, indexer | 🔜 Planned |
| **Testnet** | — | Public testnet launch | 🔜 Planned |
| **Mainnet** | — | Production launch | 🔜 Planned |

### What is a `.gitkeep` stub?

Several directories in this branch contain only a `.gitkeep` file:

```
chain/x/bridge/          ← Phase 2
chain/x/certification/   ← Phase 2
chain/x/governance-ext/  ← Phase 2
chain/x/milestone/       ← Phase 2
chain/x/oracle/          ← Phase 2
chain/x/settlement/      ← Phase 2
chain/x/validator/       ← Phase 2
```

These are intentional placeholders. The directory names are reserved, the modules are not registered in `app.go`, and their state blocks are absent from `genesis.json`. They will be implemented in Phase 2.

---

## Known Scaffold (Phase 1 Scope Boundary)

### `chain/ibcfee/` — ICS-29 local stub

`ibcFeeKeeper` is wired in all the correct places (store key, keeper field on App struct, module registered, middleware wraps the IBC transfer stack), but the implementation is a pass-through:

```go
// chain/ibcfee/fee.go
func NewIBCMiddleware(app porttypes.IBCModule, k ibcfeekeeper.Keeper) porttypes.IBCModule {
    return app  // pass-through; no fee interception
}
```

**Impact:** IBC transfers work. ICS-29 packet fee incentivization does not. This is intentional for Phase 1.

**Phase 2 fix:** Replace `chain/ibcfee/` with the real `ibc-go/v11/modules/apps/fee` package and remove the `go.mod` replace directive. The wiring in `app.go` does not change.

### `x/governance-ext` — Constitution wiring deferred

`ConstitutionContractAddr` is defined in `chain/app/wasm.go`. Passing it into `GovKeeper` requires `x/governance-ext` to have a real keeper. The module is a `.gitkeep` stub until Phase 2 — no action required now.

---

## Documentation Index

| File / Directory | Contents |
|-----------------|----------|
| [`doc/governance/genesis_parameters.md`](doc/governance/genesis_parameters.md) | Supply, staking, Bech32 prefix, chain ID reference table |
| [`doc/adr/`](doc/adr/) | All 13 Architecture Decision Records |
| [`doc/ops/runbooks.md`](doc/ops/runbooks.md) | Operational runbooks (node restart, key rotation, upgrade) |
| [`doc/ops/security_threat_model.md`](doc/ops/security_threat_model.md) | Threat model and mitigations |
| [`doc/ops/audit_engagement.json`](doc/ops/audit_engagement.json) | Audit engagement metadata |
| [`doc/testnet/onboarding.md`](doc/testnet/onboarding.md) | Validator onboarding guide |
| [`doc/testnet/stability_checklist.md`](doc/testnet/stability_checklist.md) | Pre-testnet stability checklist |
| [`doc/mainnet/chain-registry.json`](doc/mainnet/chain-registry.json) | chainlist.org / chain-registry metadata |
| [`doc/phase_0_audit_report.md`](doc/phase_0_audit_report.md) | Phase 0 audit findings |
| [`doc/phase_1_gap_analysis.md`](doc/phase_1_gap_analysis.md) | Phase 1 gap analysis (original) |
| [`final-phase-1.md`](final-phase-1.md) | Phase 1 agent fix instructions (all gaps resolved) |
| [`phase-1-final-status.md`](phase-1-final-status.md) | Full task-by-task status with before/after diff |
