# Umiña Achala: Create HCS Topics

- [Umiña Achala: Create HCS Topics](#umiña-achala-create-hcs-topics)
  - [Purpose](#purpose)
  - [The Two Topics](#the-two-topics)
  - [Topic Keys](#topic-keys)
    - [Submit Key](#submit-key)
    - [Admin Key](#admin-key)
    - [Where the Keys Come From](#where-the-keys-come-from)
    - [How a Key Is Derived](#how-a-key-is-derived)
    - [Testnet and Mainnet](#testnet-and-mainnet)
  - [Creating a Topic](#creating-a-topic)
    - [Steps](#steps)
    - [What the Script Checks](#what-the-script-checks)
  - [How the Scripts Use the Topic](#how-the-scripts-use-the-topic)
    - [Event Messages](#event-messages)
    - [Signing and the Receipt](#signing-and-the-receipt)
    - [Minting Needs the Topic](#minting-needs-the-topic)
  - [Event Catalog](#event-catalog)
  - [Status](#status)
  - [Code and Issue Reference](#code-and-issue-reference)

> **Status: Phase 2 (testnet), in progress.** The code described here is on `main` in `umina-achala-chain` (fork and upstream are the same since 2026-11-09), as of 2026-11-06, including the rename to `createTopic.js` and the Achala events. The testnet topics were created on 2026-11-10 (chain tag `v0.2.0-testnet.2`, [chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)): provenance [`0.0.11400670`](https://hashscan.io/testnet/topic/0.0.11400670) and display [`0.0.11400772`](https://hashscan.io/testnet/topic/0.0.11400772).

## Purpose

The Hedera Consensus Service (HCS) is UA's event log. Every event after the mint (sale, delivery, cutting, export, ...) is a message on an HCS topic: public, append-only and timestamped by the network. NFT metadata stays the immutable birth certificate, and the topic records everything that happens afterwards (see [create-nft-token.md](create-nft-token.md#metadata-policy-immutability)).

This document covers the topics, their keys, and how they are created and used. The event shapes themselves are defined in the data model, [chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10).

## The Two Topics

The data model uses two topics ([chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10), decided 2026-10-27):

| Topic | Env variable | Memo | Holds |
|---|---|---|---|
| Provenance | `HCS_TOPIC_ID` | `Umiña Achala provenance` | The item's history: `MINT`, `SALE`, `DELIVERED`, `TRANSFORMATION`, `EXPORT`, ... |
| Display | `HCS_DISPLAY_TOPIC_ID` | `Umiña Achala display` | What the site shows and what is for sale: `DISPLAYED`, `HIDDEN`, `LISTED`, `UNLISTED` |

They are kept apart because display events are frequent and about the site, not the item. The provenance topic stays a clean history, and the site can read the small display topic on its own. NFT metadata links only the provenance topic (`properties.hcs_compliance_topic`).

## Topic Keys

A topic has two keys, both set when it is created. Only their **public** halves are stored on the topic.

### Submit Key

The topic accepts only messages signed with the submit key. Without one, **anyone** could post fake `MINT` or `SALE` events to UA's log. This is the main reason the topic is created by a script, not by hand.

- Env variable: `HCS_SUBMIT_KEY`
- Every script that posts an event signs it with this key (see [Signing and the Receipt](#signing-and-the-receipt)).

### Admin Key

The admin key can update the topic (memo, keys, auto-renew account) or delete it. Without one, the topic could never be changed, including replacing a submit key that leaked.

- Env variable: `HCS_ADMIN_KEY`
- It signs the topic creation, and any later update or delete.

### Where the Keys Come From

**Nobody issues these keys.** Each is an ordinary ECDSA private key that UA chooses. When the topic is created, the public half is set on the topic; from then on, signing with the private half is what proves a message or an update comes from UA. So the order is always:

1. Choose the keys and put them in `.env.testnet`.
2. Create the topic with them.
3. Save the new topic ID in `.env.testnet`.

A key cannot be added to a topic later unless the topic has an admin key, which is another reason both are required.

### How a Key Is Derived

`HCS_SUBMIT_KEY` (and `HCS_ADMIN_KEY`) is not derived from anything else: not from the operator account, the token or the topic. It is a **private key**, which is just a large random number. Everything else is derived from it, in one direction:

```
random 256-bit number ──▶ private key (HCS_SUBMIT_KEY, kept secret in .env.testnet)
                               │
                               │ elliptic-curve math (secp256k1), one way only
                               ▼
                          public key ──▶ stored on the topic as its submit key
```

- **Private key → public key** is a fixed calculation, so the same private key always gives the same public key. The reverse is not feasible: knowing the topic's public key (which anyone can read on the mirror node) does not reveal the private key.
- **Signing:** a script signs each message with the private key. The network checks the signature against the public key stored on the topic, and rejects the message if it does not match.
- **Curve:** UA's scripts use **ECDSA on secp256k1**, the same key type as the UA accounts. `parseEcdsaKey` in `src/keys.js` rejects any other type, including ED25519 keys, which Hedera also supports.
- **No account needed:** a key does not have to belong to a Hedera account. It only needs to exist and be kept secret. When it is the same as `OPERATOR_KEY`, it happens to also control the operator account.

**Format in `.env.testnet`.** `parseEcdsaKey` accepts either form of the private key:

| Form | Looks like |
|---|---|
| DER, hex-encoded | starts with `3030020100300706052b8104000a04220420`, then 64 hex characters |
| Raw | 64 hex characters, optionally starting with `0x` |

Not every "DER" works. The DER that OpenSSL writes (`openssl pkcs8 -topk8 ... -outform DER`, starting `308184...`) also holds the public key, and the Hedera SDK refuses it ("invalid private key length: 135 bytes"). A PEM file (`-----BEGIN EC PRIVATE KEY-----`) is not read either. From OpenSSL, use the raw hex, as shown below.

**Any way of generating it works, if the key meets three conditions:**

1. **ECDSA on secp256k1.** Hedera also supports ED25519 keys, but UA's scripts reject them, so check what a tool generates by default.
2. **From a secure random source.** For example: the Hedera SDK (`PrivateKey.generateECDSA()`, below), a wallet that can export a private key (such as HashPack), or `openssl ecparam -name secp256k1 -genkey` converted to raw hex (option 3 below). Never a key made up by hand, and never one from a website that generates keys: anyone who has seen the private key can post events to the topic.
3. **In a format the scripts read:** DER hex or raw hex, as in the table above.

The key is then used like this: the private key goes in `.env.testnet` and never leaves the machine; `createTopic.js` sets its public key on the new topic; from then on the scripts sign each event with the private key, and the network accepts only messages whose signature matches. The topic stays tied to that key: switching to a different submit key later means updating the topic with the admin key, not just changing `.env.testnet`.

**Getting one.** One of:

1. **Reuse an existing key** (testnet only): copy the value of `OPERATOR_KEY`. Nothing new is generated; the topic's public key is then the operator account's public key.
2. **Generate a new one with the Hedera SDK**: the Hedera SDK makes a fresh random key with `PrivateKey.generateECDSA()`. Run from `umina-achala-chain`, this appends one to `.env.testnet` without printing it to the terminal:

   ```bash
   node --input-type=module -e 'import { PrivateKey } from "@hashgraph/sdk"; console.log("HCS_SUBMIT_KEY=" + PrivateKey.generateECDSA().toStringDer())' >> .env.testnet
   ```

   Remove any earlier empty `HCS_SUBMIT_KEY=` line from the file.

3. **Generate new ones with OpenSSL** (what the testnet run of 2026-11-10 does, to practise mainnet): one key for each role. Run from `umina-achala-chain`; this appends both to `.env.testnet` as raw hex, without printing them and without writing a `.pem` file:

   ```bash
   for k in HCS_SUBMIT_KEY HCS_ADMIN_KEY; do
     echo "$k=$(openssl ecparam -name secp256k1 -genkey -noout | openssl ec -outform DER 2>/dev/null | tail -c +8 | head -c 32 | xxd -p -c 64)" >> .env.testnet
   done
   ```

   **Or print them and paste them in by hand.** Each line prints one ready-to-paste line; copy each into `.env.testnet` in place of its empty `HCS_SUBMIT_KEY=` / `HCS_ADMIN_KEY=` line:

   ```bash
   echo "HCS_SUBMIT_KEY=$(openssl ecparam -name secp256k1 -genkey -noout | openssl ec -outform DER 2>/dev/null | tail -c +8 | head -c 32 | xxd -p -c 64)"
   ```

   ```bash
   echo "HCS_ADMIN_KEY=$(openssl ecparam -name secp256k1 -genkey -noout | openssl ec -outform DER 2>/dev/null | tail -c +8 | head -c 32 | xxd -p -c 64)"
   ```

   The private keys then appear on screen and stay in the terminal's scrollback (not in the shell history, which keeps only the command). Clear it afterwards (`clear`, or close the terminal), and do not paste the keys anywhere but the env file.

   `openssl ec -outform DER` writes the key in its SEC1 form, where the 32-byte private key starts at byte 8; `tail` and `head` cut it out and `xxd` turns it into 64 hex characters. Remove any earlier empty `HCS_SUBMIT_KEY=` and `HCS_ADMIN_KEY=` lines. Do not take the key from the `priv:` lines of `openssl ec -text`: they sometimes start with an extra `00`, which gives 66 characters.

   If you keep a `.pem` file (from `-out key.pem`), the same conversion is `openssl ec -in key.pem -outform DER 2>/dev/null | tail -c +8 | head -c 32 | xxd -p -c 64`. Keep the `.pem` outside the repo: only `.env*` files are git-ignored.

**Check the keys.** This lists any problem the scripts would find with them (an empty list means none), and prints only the public keys, which are safe to share:

```bash
node --input-type=module -e 'import dotenv from "dotenv"; import { envProblems } from "./src/env.js"; dotenv.config({ path: ".env.testnet" }); console.log(envProblems(process.env, ["HCS_SUBMIT_KEY", "HCS_ADMIN_KEY"]))'
```

To see the public key that will go on the topic, print only that:

   ```bash
   node --input-type=module -e 'import dotenv from "dotenv"; import { parseEcdsaKey } from "./src/keys.js"; dotenv.config({ path: ".env.testnet" }); console.log(parseEcdsaKey(process.env.HCS_SUBMIT_KEY).publicKey.toStringDer())'
   ```

A private key can only be **replaced**, never recovered: if it is lost or leaks, generate a new one and set its public key on the topic with the admin key (topic update, not built yet).

### Testnet and Mainnet

| | Testnet | Mainnet |
|---|---|---|
| `HCS_SUBMIT_KEY` | Allowed: the operator key (`OPERATOR_KEY`), following the one-key testnet decision in [chain #1](https://github.com/umina-achala/umina-achala-chain/issues/1). **Used in the testnet run of 2026-11-10:** a separate key generated with OpenSSL (option 3 above), to practise mainnet | A separate key, so posting events is not tied to paying for transactions |
| `HCS_ADMIN_KEY` | Allowed: the operator key. **Used in the testnet run of 2026-11-10:** a separate key generated with OpenSSL | A separate key, ideally multi-signature (see [chain #32](https://github.com/umina-achala/umina-achala-chain/issues/32)) |

With a separate submit key, `HCS_SUBMIT_KEY` must stay in `.env.testnet` for every script that records an event: without it the operator's signature alone does not match the topic's submit key, and the event is rejected with `INVALID_SIGNATURE`.

Proposed on [chain #9](https://github.com/umina-achala/umina-achala-chain/issues/9#issuecomment-6663344252) and [chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17#issuecomment-6663343568). **Mainnet keys: decision not made yet.**

## Creating a Topic

Hedera docs: [Create a topic](https://docs.hedera.com/native/consensus/create-topic), [Submit a message](https://docs.hedera.com/native/consensus/submit-message). The script's options are in [scripts.md](scripts.md#createtopicjs).

### Steps

Run from `umina-achala-chain`. Creating a topic writes to Hedera, so the project owner runs it.

1. In `.env.testnet`, set `HCS_SUBMIT_KEY` and `HCS_ADMIN_KEY` (on testnet, both the same as `OPERATOR_KEY`).
2. Check the settings without sending anything:

   ```bash
   node src/topic/createTopic.js provenance --dry-run
   ```

3. Create the provenance topic:

   ```bash
   node src/topic/createTopic.js provenance
   ```

4. Save the printed ID in `.env.testnet` as `HCS_TOPIC_ID`.
5. Repeat with `display` when the display topic is needed, and save it as `HCS_DISPLAY_TOPIC_ID`.

The full usage is `node src/topic/createTopic.js <provenance|display> [env-file] [--force] [--dry-run]`. The env file defaults to the network's own (`.env.testnet`; `NETWORK` chooses the network, testnet when unset), and a file given must name the network.

### What the Script Checks

Before sending anything:

- **The topic kind** is given: `provenance` or `display`. A topic is never created without saying which.
- **The env file** is a testnet file. The script refuses any other, so mainnet credentials are never sent to testnet.
- **`OPERATOR_ID`** looks like `0.0.N`, and **`OPERATOR_KEY`, `HCS_SUBMIT_KEY` and `HCS_ADMIN_KEY`** are present and are ECDSA private keys.
- **No existing topic is replaced.** A real run looks up the current value of the env variable (`HCS_TOPIC_ID` or `HCS_DISPLAY_TOPIC_ID`) on the mirror node, and refuses if it names a topic that exists and is not deleted. A placeholder that does not exist, such as `0.0.1234`, does not block. `--force` skips this check.

`--dry-run` prints the topic settings (kind, memo, the variable to save the ID in, auto-renew account, and whether each key is the operator key) without any key material and without touching the network or the mirror node.

The topic is created with the operator as its **auto-renew account**, so it does not expire. The operator pays the fee, and the admin key signs.

## How the Scripts Use the Topic

### Event Messages

Every event follows the envelope from [chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10), built by `buildEvent` in `src/hcs.js`:

| Field | Example | Notes |
|---|---|---|
| `event` | `"SALE"` | Event type |
| `schema_version` | `1` | Bumped when a message shape changes |
| `stone_id` | `"UMINA-2026-CH-03"` | Left out when the script only knows the serial (sales) |
| `token_id` | `"0.0.1234"` | |
| `serials` | `["7"]` | |
| `actor_id` | `"0.0.5678"` | The account that submitted the event |
| `occurred_at` | `"2026-10-27T15:00:00Z"` | When it happened, UTC to the second; the consensus timestamp records when it was logged |
| `ref` | `"0.0.5678@1761577200.123"` | The related Hedera transaction: the mint for `MINT`, the sale for `SALE` |

Event-specific fields follow the envelope. For example, `MINT` adds `ipfs_cid`, and `SALE` adds `seller_id`, `buyer_id`, `payment`, `sold_for` and `currency`. A message larger than 1 KB (one HCS message) is refused before sending; larger content goes to IPFS and the event carries the CID.

Everything on the topic is public and permanent: account IDs, dates and hashes only, no personal data.

### Signing and the Receipt

`submitEvent` in `src/hcs.js` is the only way the scripts post an event ([chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17)):

- It signs the message with `HCS_SUBMIT_KEY` when that is set. Until the topic exists, the key is optional in the mint and sale scripts; one that is set but not ECDSA is rejected.
- It **waits for the receipt**. A message the network rejects at consensus (wrong signature, deleted or expired topic) is reported as a failure, not as a written record. Before #17, the scripts never checked, so a rejected message looked like a success.

When an event fails after the on-chain action succeeded (the NFT is minted, or the sale has settled), the script says so and prints what is needed to record it by hand.

### Minting Needs the Topic

Before pinning anything, a real mint reads the provenance topic from the mirror node ([chain #20](https://github.com/umina-achala/umina-achala-chain/issues/20)):

- If the topic does not exist, the mint is refused, so an NFT is never minted with nowhere to record its `MINT` event. Minting was blocked on purpose until the provenance topic was created (2026-11-10).
- If the topic already has a `MINT` event for the same `stone_id`, the mint is refused unless `--force` is given.

How a missing topic is detected: the testnet mirror node answers `200` with an empty message list for the messages of a topic that does not exist; only `/api/v1/topics/<id>` answers `404`. So `topicEvents` in `src/mirror.js` looks the topic itself up first, and treats a missing or deleted topic as not found (fixed 2026-11-05, [chain #20](https://github.com/umina-achala/umina-achala-chain/issues/20)).

## Event Catalog

What each script records in the provenance topic, as of 2026-11-06. Every event has the [envelope](#event-messages); the catalog itself is kept on [chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10).

| Event | Recorded by | When | Event fields | `ref` |
|---|---|---|---|---|
| `COMPLIANCE` | `mint/mintNFT.js` | Before the metadata is pinned; its transaction ID becomes `compliance_proof_hcs` in the metadata ([chain #14](https://github.com/umina-achala/umina-achala-chain/issues/14)). `serials` is `[]` | Umiña: `stone_type`, `vendor_ruc`, `mining_concession`, `reinfo_id`, `hs_code`. Achala: `piece_type`, `artisan_rna`, `vendor_ruc`, `hs_code`, `from_stone_ids` | |
| `MINT` | `mint/mintNFT.js` | After the mint | `ipfs_cid`; an Achala adds `from_stone_ids`, which records its stones as mounted ([chain #16](https://github.com/umina-achala/umina-achala-chain/issues/16)) | the mint transaction |
| `ISSUED_TO_OWNER` | `consignment/consignToMerchant.js` | After the transfer to the merchant and the allowance | `to_id`, `allowance_ref` | the transfer |
| `SALE` | `consignment/sellConsigned.js`, `sale/sellOwnItem.js`, `record/recordEvent.js sale` | After an on-chain sale, or to record one made elsewhere (checked on the mirror node; a second `SALE` for the same `ref` is refused) | `seller_id`, `buyer_id`, `payment`, `sold_for`, `currency`, optional `payment_ref`; the consigned sale adds the split in tinybars | the transfer that moved the NFT |
| `DELIVERED` | `record/recordEvent.js delivered` | The buyer has the piece (in person so far) | `buyer_id`, `method`, `delivered_at`, `proof` (`signed` or `none`), `proof_hash` when signed | the `SALE`'s transaction |
| `TRANSFER` | `consignment/endConsignment.js` | An unsold consigned item goes back to the Treasury | `from_id`, `to_id`, `reason: "consignment_ended"` | the return transfer |
| `BURN` | `token/adminToken.js wipe` / `burn` | A serial is wiped from a holder or burned in the Treasury; also the Achala of a piece taken apart: its `BURN` unmounts its stones (their Umiña NFTs are not burned), which can be mounted again | `reason`, `method`, `from_id` for wipe | the wipe or burn |

Mounting and unmounting are recorded only in the Achala's own events (`MINT` and `BURN`); the stones get no events of their own (decided 2026-11-06).

**Still to settle before mainnet**, since HCS messages cannot be changed once written (on testnet, a change after the run, [chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22), only means creating a fresh topic):

- `stone_id` in every event: today only `COMPLIANCE` and `MINT` always carry it; `SALE`, `ISSUED_TO_OWNER`, `TRANSFER` and `BURN` name the item by `token_id` and serial only ([chain #25](https://github.com/umina-achala/umina-achala-chain/issues/25)).
- Small fixes: a `COMPLIANCE` left with no `MINT` by a failed mint, `occurred_at` as the consensus time everywhere, the consignment event names ([chain #27](https://github.com/umina-achala/umina-achala-chain/issues/27)).

The last Phase 2A code is written, on `main` in the chain repo (2026-11-06; upstream since 2026-11-09): an Achala's sale moves it and its stones in one transfer and records one `SALE` with `from_stone_ids`, `umina_token_id` and `umina_serials` (`sellOwnItem.js --token achala`; `recordEvent.js --token achala` for `SALE` and `DELIVERED` made elsewhere, [chain #26](https://github.com/umina-achala/umina-achala-chain/issues/26)); and a stone mounted in an Achala that still exists is refused by the sale, consignment, burn and wipe scripts, which read the topic first ([chain #24](https://github.com/umina-achala/umina-achala-chain/issues/24)).

Planned, not recorded by any script yet: `TRANSFORMATION` (cutting or polishing), `EXPORT`, the display-topic events (`LISTED`, `UNLISTED`, `DISPLAYED`, `HIDDEN`) and the rest of the catalog in [chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10).

## Status

- The topic script is written and tested, on `main` ([chain #9](https://github.com/umina-achala/umina-achala-chain/issues/9)); renamed from `createUminaTopic.js` to `createTopic.js`, since the topics serve both token classes.
- Testnet topics created by the project owner on 2026-11-10 ([chain #22](https://github.com/umina-achala/umina-achala-chain/issues/22)): provenance [`0.0.11400670`](https://hashscan.io/testnet/topic/0.0.11400670) and display [`0.0.11400772`](https://hashscan.io/testnet/topic/0.0.11400772), about 0.46 HBAR each. Both have the same separate submit and admin keys (ECDSA, generated with OpenSSL, not the operator key). The display topic stays empty until a script or the site posts to it.
- Mainnet keys are not decided.

## Code and Issue Reference

Code is in `umina-achala-chain`.

| Topic | Code | Issue |
|---|---|---|
| Create a topic | `src/topic/createTopic.js` | [chain #9](https://github.com/umina-achala/umina-achala-chain/issues/9) |
| Topic env checks, transaction, existing-topic check | `src/topic/lib.js` (`validateTopicEnv`, `buildTopicTransaction`, `existingTopicProblem`) | [chain #9](https://github.com/umina-achala/umina-achala-chain/issues/9) |
| Mirror node lookups | `src/mirror.js` (`topicInfo`, `topicEvents`) | [chain #9](https://github.com/umina-achala/umina-achala-chain/issues/9), [#20](https://github.com/umina-achala/umina-achala-chain/issues/20) |
| Event envelope, submit-key signing, receipt | `src/hcs.js` (`buildEvent`, `submitEvent`, `submitKeyProblem`) | [chain #17](https://github.com/umina-achala/umina-achala-chain/issues/17) |
| Event shapes and the two topics | | [chain #10](https://github.com/umina-achala/umina-achala-chain/issues/10) |
| Env variables | `.env.example` (`HCS_TOPIC_ID`, `HCS_DISPLAY_TOPIC_ID`, `HCS_SUBMIT_KEY`, `HCS_ADMIN_KEY`) | |
