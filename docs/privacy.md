# Privacy-Preserving Transactions

NSSA supports both public and privacy-preserving transactions. Programs written in SPEL work with both — the same program code runs inside a privacy-preserving ZK circuit when invoked privately.

## Private Account Keys

Each private account has a key chain derived from a single secret:

| Key | Description |
|---|---|
| **NullifierSecretKey (nsk)** | 32 bytes, secret. Proves account ownership. Stored in the wallet. |
| **NullifierPublicKey (npk)** | Derived from nsk via hash. Acts as the account's identity. |
| **ViewingPublicKey (vpk)** | Secp256k1 point. Allows decrypting account data without full ownership. |

**Private account IDs** are derived as `SHA256("/LEE/v0.3/AccountId/Private/" || npk)` — an observer cannot link an npk to its account ID without knowing the npk.

## How Private Transactions Work

### Visibility Mask

Each account in a transaction is tagged with a visibility level:

| Tag | Meaning |
|---|---|
| `0` | Public — fully visible on-chain |
| `1` | Private + authenticated — signer, wallet holds the nsk |
| `2` | Private + unauthenticated — recipient, wallet only knows npk + vpk |

### Transaction Structure

A privacy-preserving transaction contains:

```
Message {
    public_account_ids:           []       // only public accounts listed
    public_post_states:           []       // visible state changes (public accounts only)
    encrypted_private_post_states: [...]   // private accounts encrypted with recipient's vpk
    new_commitments:              [...]    // added to on-chain Merkle tree
    new_nullifiers:               [...]    // proves old state was consumed
}
+ ZK Proof (RISC Zero receipt)
```

### Commitments

When a private account's state changes, a **commitment** is computed and added to an on-chain Merkle tree:

```
Commitment = SHA256(npk || program_owner || balance || nonce || SHA256(data))
```

The commitment binds the account state without revealing it. The Merkle tree provides membership proofs for spending.

### Nullifiers

To spend (update) a private account, the wallet produces a **nullifier** from the nsk and the old commitment. This proves the old state was consumed without revealing _which_ commitment it corresponds to.

The sequencer tracks all nullifiers and rejects duplicates — this prevents double-spending.

### Encrypted Post-States

Updated private account states are encrypted with the recipient's viewing public key. Only the recipient (or anyone with the viewing key) can decrypt and read the new balance and data.

### View Tags

```
ViewTag = SHA256("/NSSA/v0.2/ViewTag/" || npk || vpk)[0]   // first byte only
```

A lightweight 1-byte filter. Wallets check the view tag before attempting full decryption, filtering out ~99.6% of irrelevant transactions without any expensive crypto operations.

### Sequencer Validation

The sequencer verifies:

1. The ZK proof is valid (RISC Zero receipt verification)
2. All new commitments are fresh (not seen before)
3. All nullifiers are fresh (no double-spend)
4. Public account signatures and nonces are correct
5. Then: updates public state, appends commitments to the Merkle tree, records nullifiers

## Privacy Guarantees

| Transfer Type | Sender Visible | Receiver Visible | Amount Visible |
|---|---|---|---|
| Public → Public | ✅ Yes | ✅ Yes | ✅ Yes |
| Public → Private | ✅ Yes (debited) | ❌ No (encrypted) | Partially (public debit visible) |
| Private → Private | ❌ No | ❌ No | ❌ No |
| Private → Public | ❌ No (nullifier only) | ✅ Yes (credited) | Partially (public credit visible) |

A **private → private** transaction reveals _nothing_ about the sender, receiver, or amount. An observer only sees that some old commitments were consumed and new encrypted states appeared.

## Wallet Operations

### Create a Private Account

```bash
wallet account new private
```

This generates nsk, npk, and vpk, and stores them in the wallet.

### Initialize (Required Before First Use)

```bash
wallet auth-transfer init --account-id Private/<base58>
```

This registers the account's commitment in the Merkle tree. **Must be done before any transfers**, even for pre-funded genesis accounts.

### Sync Private State

```bash
wallet account sync-private
```

Scans the blockchain for new commitments that belong to your accounts (using view tags for fast filtering, then full decryption). **Always sync before making private transfers** — the wallet needs up-to-date membership proofs.

### Transfer Tokens

```bash
# Public → Private
wallet auth-transfer send \
  --from Public/<sender_base58> \
  --to Private/<receiver_base58> \
  --amount 1000

# Private → Private (fully private)
wallet auth-transfer send \
  --from Private/<sender_base58> \
  --to Private/<receiver_base58> \
  --amount 300

# Private → Public
wallet auth-transfer send \
  --from Private/<sender_base58> \
  --to Public/<receiver_base58> \
  --amount 500
```

## Important Notes

1. **Always sync before private transfers.** Without sync, the wallet lacks membership proofs and the privacy circuit will reject the transaction with "Found new private account with non default values."

2. **Always init before first use.** Even pre-funded genesis accounts need `auth-transfer init` — their balances exist but their commitments may not be in the Merkle tree.

3. **Private transfers are CPU-intensive.** The wallet generates a RISC Zero ZK proof locally for each private transaction.

4. **The `auth-transfer` program** is the built-in program for native token transfers with variable privacy. Custom SPEL programs can also participate in private transactions — the same program code runs inside the privacy-preserving circuit automatically.

5. **Viewing keys enable selective disclosure.** You can share a viewing public key to let a third party (auditor, regulator) read your private account balances without giving them spending authority.
