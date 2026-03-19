# Local Development Setup (Without Docker)

This guide covers building SPEL programs and running a local sequencer without Docker — useful for containerized environments, CI pipelines, or machines where Docker isn't available.

## Building Without Docker

The default `make build` runs `cargo risczero build`, which requires Docker to produce deterministic guest binaries. For development and testing, you can build locally using the RISC Zero toolchain directly.

### Prerequisites

1. **Rust** (stable + nightly):

   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   rustup install nightly
   ```

2. **rzup** (RISC Zero toolchain manager — installs the `riscv32im-risc0-zkvm-elf` target):

   ```bash
   curl -L https://risczero.com/install | bash
   rzup install
   ```

### Building

Instead of `make build`, run:

```bash
cargo build --release
```

The project's `methods/build.rs` calls `risc0_build::embed_methods()`, which defaults to local (non-Docker) builds when Docker isn't detected. This uses the rzup-installed RISC Zero Rust toolchain to compile the guest binary.

### Binary Path Difference

| Build Method | Binary Path |
|---|---|
| Docker (`make build`) | `methods/guest/target/riscv32im-risc0-zkvm-elf/docker/<name>.bin` |
| Local (rzup) | `target/riscv-guest/<name>-methods/<name>-guest/riscv32im-risc0-zkvm-elf/release/<name>.bin` |

Update your `-p` flag accordingly when calling `lez-cli` or `make cli`.

> **Note:** Local builds are _not_ deterministic/reproducible. The resulting binaries are functionally identical but will have different ImageIDs than Docker builds. Use Docker builds for production deployments where reproducibility matters.

### Memory Considerations

RISC Zero guest compilation is memory-intensive. On memory-constrained hosts (containers, small VMs), limit parallelism:

```bash
cargo build --release --jobs 2
```

Or set environment variables:

```bash
export CARGO_BUILD_JOBS=2
```

---

## Setting Up a Local Sequencer

The sequencer and wallet come from [logos-blockchain/lssa](https://github.com/logos-blockchain/lssa).

### Build

```bash
git clone https://github.com/logos-blockchain/lssa.git ~/lssa
cd ~/lssa
cargo build --release -p sequencer_runner -p wallet --jobs 2
```

### Start the Sequencer

```bash
cd ~/lssa
./target/release/sequencer_runner sequencer_runner/configs/debug/sequencer_config.json &
```

The sequencer runs on port **3040** by default. Verify it's up:

```bash
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:3040
# Should return 405 (alive, no GET handler)
```

### Configure the Wallet

```bash
export PATH=~/lssa/target/release:$PATH
export NSSA_WALLET_HOME_DIR=~/lssa/wallet/configs/debug

wallet check-health    # Should print "All looks good!"
wallet account ls      # List preconfigured accounts
```

### Deploy and Interact

```bash
cd ~/my-program

# Deploy
wallet deploy-program <path-to-binary>

# Interact via lez-cli
lez-cli -i my-program-idl.json -p <path-to-binary> --help
lez-cli -i my-program-idl.json -p <path-to-binary> initialize --owner-account <SIGNER_BASE58>
```

---

## Version Compatibility

`lez-cli` pins to a specific lssa git revision in its `Cargo.toml`. If the wallet config format has changed between that revision and the current lssa HEAD, you may see deserialization errors like:

```
missing field `seq_poll_timeout_millis` at line X column Y
```

**Fix:** Check which lssa revision lez-cli is pinned to, and use wallet configs from that version:

```bash
# Find the pinned rev
grep 'logos-blockchain/lssa' ~/spel/lez-cli/Cargo.toml

# Export compatible config
cd ~/lssa
git show <pinned-rev>:wallet/configs/debug/wallet_config.json > /path/to/compatible_config.json
export NSSA_WALLET_HOME_DIR=/path/to/config-dir
```
