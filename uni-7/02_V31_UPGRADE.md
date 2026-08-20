# uni-7 v31 upgrade (draft)

> **Preparation status:** the v31 release tag, release commit, upgrade height, and UTC estimate are not final. Do not schedule or stage this upgrade from this draft.

uni-7 will upgrade from Juno `v30.0.0` to `v31.0.0` after the release candidate and schedule are finalized.

- Chain ID: `uni-7`
- Current version: `v30.0.0`
- Target version: `v31.0.0` (**release pending**)
- Upgrade plan name: **`v31`**
- Upgrade height: **TBD**
- Release commit: **TBD**
- Explorer: **TBD after height selection**
- Candidate integration: <https://github.com/CosmosContracts/juno/pull/1223>

This is a coordinated consensus upgrade. The current v31 candidate keeps the v30 store architecture and declares no store additions or deletions. Its upgrade handler migrates module versions and initializes the default Clock and CW Hooks contract caps when legacy state contains zero values.

The current candidate moves to Cosmos SDK v0.53.8, wasmd v0.61.14, wasmvm v3.0.7, IBC-Go v10.7.0, and CometBFT v0.38.25. It also contains FeePay, feegrant, wallet gas-simulation, transaction gas-limit, export/import, state-sync, and release-pipeline fixes. These details must be rechecked against the immutable release tag before this guide is submitted upstream.

## Release gates

Do not select the halt height or announce the upgrade until all gates pass:

- [ ] the v31 integration PR is merged and its required checks pass;
- [ ] the immutable `v31.0.0` tag and release commit exist;
- [ ] the exact source tag builds with the release's pinned Go toolchain;
- [ ] `junod version --long` reports the expected version, commit, and Cosmos SDK version;
- [ ] wasmvm v3.0.7 linkage is present and resolves on the validator host;
- [ ] published release assets and provenance are inspected;
- [ ] the height and UTC estimate are calculated from fresh uni-7 block-time samples;
- [ ] live RPC/API endpoints and post-upgrade commands are verified.

## Before the halt

1. Back up validator state, the current v30 binary, and the newest signing state.
2. Ensure the node is healthy and fully synced on `v30.0.0`.
3. Keep the existing fee-market-compatible `minimum-gas-prices` configuration.
4. Build and stage the final v31 release before the halt.
5. Record the staged binary's `version --long` output and SHA-256 checksum.

## Build from source

Replace the placeholders below only after the immutable release exists:

```bash
RELEASE_TAG="v31.0.0"
RELEASE_COMMIT="TBD"

cd "$HOME"
git clone https://github.com/CosmosContracts/juno.git juno-v31
cd juno-v31
git checkout "$RELEASE_TAG"

test "$(git rev-list -n 1 "$RELEASE_TAG")" = "$RELEASE_COMMIT"
LEDGER_ENABLED=false make build

./bin/junod version --long | grep "cosmos_sdk_version\|commit\|version:"
sha256sum ./bin/junod
file ./bin/junod
ldd ./bin/junod || true
```

The final guide will include the exact expected three-line version output and the verified release-artifact path.

## Stage with Cosmovisor

The directory name must match the on-chain plan name exactly: `v31`.

```bash
export DAEMON_HOME="${DAEMON_HOME:-$HOME/.juno}"
mkdir -p "$DAEMON_HOME/cosmovisor/upgrades/v31/bin"
install -m 0755 "$HOME/juno-v31/bin/junod" \
  "$DAEMON_HOME/cosmovisor/upgrades/v31/bin/junod"

"$DAEMON_HOME/cosmovisor/upgrades/v31/bin/junod" version --long
sha256sum "$DAEMON_HOME/cosmovisor/upgrades/v31/bin/junod"
```

Do not replace the running v30 binary before the halt unless Cosmovisor is managing the version switch.

## Post-upgrade verification

After the final height is set, verify that uni-7 produces blocks beyond it and run:

```bash
RPC="https://juno-testnet-rpc.cogwheel.zone"

junod status --node "$RPC"
junod query upgrade applied v31 --node "$RPC" --output json
junod query upgrade module-versions --node "$RPC" --output json
junod query feemarket params --node "$RPC" --output json
junod query feemarket gas-price ujunox --node "$RPC" --output json
junod query cw-hooks params --node "$RPC" --output json
junod query clock params --node "$RPC" --output json
```

Acceptance criteria:

- the applied plan is `v31` at the selected height;
- validators agree on the app hash and continue producing blocks;
- Clock and CW Hooks contract caps are non-zero after migration;
- a normal wallet transfer simulates and delivers with adequate gas;
- feegrant and FeePay transactions simulate and deliver with correct sender accounting;
- IBC relayers can relay and update clients using fee grants;
- existing CosmWasm contracts can be queried and executed;
- export/import, snapshot, and state-sync operations remain healthy;
- no wasmvm linkage, migration, overflow, store-loader, or consensus errors appear in node logs.
