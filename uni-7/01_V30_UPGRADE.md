# uni-7 v30 upgrade

uni-7 will upgrade to Juno [`v30.0.0`](https://github.com/CosmosContracts/juno/releases/tag/v30.0.0) at block **`16000000`**. At the recent 2.60-second average block time, the halt is estimated for **2026-07-20 around 13:50 UTC**. The block height takes precedence over the estimate.

- Chain ID: `uni-7`
- Current version: `v29.0.0`
- Target version: `v30.0.0`
- Upgrade plan name: **`v30`**
- Upgrade height: **`16000000`**
- Release commit: `c0b3a8d258d52d16e5bc39a75168a99aab9d098e`
- Explorer: <https://testnet.juno.valopers.com/blocks/16000000>

This is a consensus-breaking upgrade. v30 moves to Cosmos SDK v0.53.7, wasmd v0.61.11, wasmvm v3.0.4, IBC-Go v10.6.0, and CometBFT v0.38.23. It adds the `feemarket` and `votingsnapshot` stores and deletes the `globalfee`, `crisis`, `params`, `nft`, `feeibc`, and `interchainquery` stores.

## Before the halt

1. Back up the validator state and current binary.
2. Ensure the node is healthy and fully synced on v29.0.0.
3. Set `minimum-gas-prices = "0.075ujunox"` in `~/.juno/config/app.toml`, or leave it empty so the on-chain fee market sets the floor. Do not retain a lower non-empty value.
4. Build and stage v30 before the halt. Go 1.25.x is required.
5. Record the staged binary's `version --long` output and SHA-256 checksum. Every validator should run the exact release tag/commit above.

The GitHub binary-release workflow did not publish a release asset for v30.0.0, so build the binary from the immutable source tag rather than downloading an unverified binary from a third party.

## Build from source

```bash
cd "$HOME"
git clone https://github.com/CosmosContracts/juno.git juno-v30
cd juno-v30
git checkout v30.0.0

# Must print c0b3a8d258d52d16e5bc39a75168a99aab9d098e
test "$(git rev-list -n 1 v30.0.0)" = "c0b3a8d258d52d16e5bc39a75168a99aab9d098e"

LEDGER_ENABLED=false make build
./bin/junod version --long
sha256sum ./bin/junod
```

The repository Docker build produces a statically linked wasmvm v3 binary. A locally built binary may instead link dynamically. Check it before staging:

```bash
file ./bin/junod
ldd ./bin/junod || true
```

If `ldd` lists `libwasmvm`, ensure the matching wasmvm v3.0.4 library is installed on the validator host before restart. Do not use a wasmvm v2 shared library with v30.

## Stage with Cosmovisor

The directory name must match the on-chain plan name exactly: `v30`.

```bash
export DAEMON_HOME="${DAEMON_HOME:-$HOME/.juno}"
mkdir -p "$DAEMON_HOME/cosmovisor/upgrades/v30/bin"
install -m 0755 "$HOME/juno-v30/bin/junod" \
  "$DAEMON_HOME/cosmovisor/upgrades/v30/bin/junod"

"$DAEMON_HOME/cosmovisor/upgrades/v30/bin/junod" version --long
sha256sum "$DAEMON_HOME/cosmovisor/upgrades/v30/bin/junod"
```

Confirm `version: v30.0.0` and commit `c0b3a8d258d52d16e5bc39a75168a99aab9d098e` before the halt.

## Manual upgrade

If Cosmovisor is not being used, wait for the chain to halt at block 16000000, then:

```bash
sudo systemctl stop junod
cp "$HOME/go/bin/junod" "$HOME/go/bin/junod-v29.0.0"
install -m 0755 "$HOME/juno-v30/bin/junod" "$HOME/go/bin/junod"
"$HOME/go/bin/junod" version --long
sha256sum "$HOME/go/bin/junod"
sudo systemctl start junod
journalctl -u junod -f --no-hostname
```

Do not replace the running v29 binary before the halt unless Cosmovisor is managing the version switch.

## Post-upgrade verification

Verify that the chain produces blocks beyond 16000000, then run:

```bash
RPC="https://rpc-uni.junonetwork.io:443"

junod status --node "$RPC"
junod query upgrade applied v30 --node "$RPC" --output json
junod query upgrade module-versions --node "$RPC" --output json
junod query feemarket params --node "$RPC" --output json
junod query feemarket gas-price ujunox --node "$RPC" --output json
junod query cw-hooks params --node "$RPC" --output json
```

Also verify the new REST surfaces when an API endpoint is available:

```bash
API="https://lcd-uni.junonetwork.io"
UPGRADE_HEIGHT=16000000
DELEGATOR="<existing-juno-address>"

curl -fsS "$API/juno/feemarket/v1/params" | jq
curl -fsS "$API/juno/feemarket/v1/state" | jq
curl -fsS "$API/juno/feemarket/v1/gas_price/ujunox" | jq
curl -fsS "$API/juno/votingsnapshot/v1/params" | jq
curl -fsS "$API/juno/votingsnapshot/v1/voting_power/$DELEGATOR/$UPGRADE_HEIGHT" | jq
curl -fsS "$API/juno/votingsnapshot/v1/total_voting_power/$UPGRADE_HEIGHT" | jq
```

Acceptance criteria:

- the applied plan is `v30` at block 16000000;
- validators agree on the app hash and continue producing blocks;
- the feemarket is enabled for `ujunox` with a positive gas price;
- cw-hooks params are readable and the failure-removal threshold is `3`;
- voting-snapshot returns sensible backfilled voting power for a pre-upgrade delegator;
- an existing CosmWasm contract can be queried and executed;
- existing IBC connections/channels still work and packets sent across the boundary acknowledge or time out cleanly;
- no `undefined symbol: wasmvm_*`, CGO ABI, store-loader, or migration errors appear in node logs.

Because v30 removes the `feeibc` and `interchainquery` stores, any residual ICS-29 fee escrow or async-ICQ state will not survive the upgrade. Operators should record active IBC/ICQ state before the halt and report unexpected state to the Juno team immediately.
