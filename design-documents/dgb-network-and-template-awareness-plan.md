# dgbminer: DigiByte network, algorithm and DigiDollar awareness

**Status:** Implemented on `dgb-network-and-template-awareness`. Built natively on Ubuntu (`build.sh`, GCC 15.2) and live-tested on testnet26 on 2026-09-27. Everything is verified except the oracle output, which is waiting on a network that offers a bundle. See §10. See §8.
**Repo:** `mmfpsolutions/dgbminer` (public fork of `DigiByte-Core/dgbminer`)
**Branch:** `dgb-network-and-template-awareness`
**Upstream PR:** not planned. These changes stay in the MMFP fork.

## 1. Why

dgbminer is a CPU miner for DigiByte's five algorithms. GoSlimStratum used it to verify GSS's new
Odocrypt support (GSS 5.3.0). That work found three gaps:

1. **The Odo key interval is hard-coded to 1 day**, so Odo mining is broken on mainnet 9 days in 10.
2. **Solo templates don't name an algorithm**, so the node picks one from `algo=` in `digibyte.conf`.
   One node can't serve several dgbminer instances on different algorithms.
3. **Solo templates ignore DigiDollar.** dgbminer never requests the `digidollar-oracle` rule, so its
   blocks never carry the oracle commitment.

Gaps 2 and 3 only affect **solo mining** (`--no-stratum`, `getblocktemplate`). In stratum mode the pool
builds the template and coinbase. Gap 1 affects **both** modes, because the miner computes the Odo
key itself.

## 2. Gap 1: Odo key interval

### Problem

`algo/odo/odo.cpp`:

```cpp
static constexpr uint32_t OdoShapechangeInterval = 1*24*60*60;
uint32_t key = OdoKey(OdoShapechangeInterval, data[17]);   // data[17] = nTime
```

The interval is a per-network consensus parameter (`nOdoShapechangeInterval` in DigiByte's
`src/kernel/chainparams.cpp`):

| Network | Interval |
|---|---|
| mainnet | 10 days |
| testnet (testnet26) | 1 day |
| regtest | 10 days |

Every 10-day boundary is also a day boundary. So on mainnet the 1-day key is right only on the first
day of each 10-day period. On the other 9 days every hash is computed under the wrong key, and the node
rejects every block. A pool rejects the shares as low difficulty.

**Proven on real data** (from the GSS work): four mainnet Odo blocks, each from a different 10-day
period, all match the node's `pow_hash` with the 10-day key and all fail with the 1-day key.

### Fix

- Add a **`--testnet`** option, with a new `case 10xx:` in `cpu-miner.c`'s option parser, a global
  `bool opt_testnet`, and a line in the usage text.
- `scanhash_odo` uses `opt_testnet ? 86400 : 864000`.
- The **default is 10 days** (mainnet and regtest). Mainnet is where real users mine, and it's the case
  that's broken today.
- The startup log states the interval in use, for example `Odo key interval: 10 days (mainnet/regtest)`,
  so a wrong setting is visible straight away.

**Behaviour change:** anyone mining Odo on testnet26 today must add `--testnet`. This goes in the README.
It's the same flag and meaning as odo-miner's (MentalCollatz/odo-miner) stratum proxy.

## 3. Gap 2: algorithm-aware templates

### Problem

The request is a compile-time constant with no algorithm (`cpu-miner.c` around line 1509):

```c
#define GBT_RULES "[\"segwit\"]"
static const char *gbt_req =
   "{\"method\": \"getblocktemplate\", \"params\": [{\"capabilities\": "
   GBT_CAPABILITIES ", \"rules\": " GBT_RULES "}], \"id\":0}\r\n";
```

`gbt_lp_req` (the long-poll request) is the same.

### Fix

DigiByte's `getblocktemplate` takes the algorithm as its **second positional parameter**. GSS does
exactly this in production (`pkg/coin/digibyte/rpc.go`):

```json
{"method":"getblocktemplate","params":[{"capabilities":[…],"rules":["segwit","digidollar-oracle"]},"odo"],"id":0}
```

- Build both requests **at runtime**, once, after option parsing. The algorithm is only known then.
- Pass `algo_names[opt_algo]`. dgbminer's names already match DigiByte's: `sha256d`, `scrypt`, `skein`,
  `qubit`, `odo`.
- Only for the five DigiByte algorithms. Any other `-a` value keeps today's request unchanged, since
  dgbminer inherits other algorithms from cpuminer-opt.
- The node then returns that algorithm's nBits and algorithm version bits, whatever `algo=` says in
  `digibyte.conf`.

The README's "set `algo=` in `digibyte.conf`" note becomes optional.

## 4. Gap 3: DigiDollar oracle commitment

### Problem

dgbminer requests only `segwit`, so the node never returns `default_oracle_commitment`. Its blocks are
still valid: a testnet26 Odo block without the commitment was accepted, at height 442602. But they
carry no oracle data.

### Fix

Mirror GSS's proven implementation (`pkg/coin/digibyte/coinbase.go`, `appendOracleCommitment`,
shipped in GSS 5.1.4 and live on mainnet):

1. Add `"digidollar-oracle"` to the requested rules (DigiByte algorithms only). This is safe on every
   node version and activation state. A node that doesn't know the rule, or where it isn't active,
   returns a normal template with no commitment.
2. When the template has `default_oracle_commitment`, add a **zero-value output** whose scriptPubKey is
   that hex **verbatim**. It already starts with `0x6a`, OP_RETURN.
3. **Order:** payout output, then the oracle commitment, then the witness commitment last, following the
   BIP141 convention that GSS also follows.
4. The **output count** becomes `1 + oracle + segwit`. Today it's `segwit ? 2 : 1`.
5. Encode the script length as a **compact size**. Today it's a single byte, which is fine for the witness
   commitment's fixed 38 bytes but not guaranteed for the oracle script.

### Trap: the coinbase buffer

The coinbase is built in a fixed buffer, `cbtx = (uchar*) malloc(256)`. Current worst case, roughly:

- base, height and payout output: about 95 bytes
- witness output: 47 bytes
- lock time: 4 bytes
- the scriptSig extension (`--coinbase-sig` and `coinbaseaux`, moved into place with `memmove`): up to 102 bytes

That's already close to 256. **Adding an oracle output of about 100 bytes could overflow it**, silently
corrupting memory. The fix is to size the buffer from the actual inputs: a base allowance plus the oracle
script length plus the maximum scriptSig extension. Also check before each append, and fail with an error
rather than overrun.

## 5. What doesn't change

- Stratum mode's job, share and submit handling. Only the Odo key (gap 1) applies there.
- Every non-DigiByte algorithm dgbminer inherits from cpuminer-opt.
- Hashing code, apart from the Odo interval.

## 6. Testing

**Gap 1:**
- Build a small check that runs `scanhash_odo`'s key and hash path on the known vectors
  (`goslimstratum/design-documents/multi-algo-designs/dgb-odo-blocks-examples.md`). The four mainnet
  blocks must match with the default, and the testnet26 block with `--testnet`.
- **Live:** mine Odo on testnet26 with `--testnet`, solo and through GSS. Blocks must be accepted.

**Gaps 2 and 3, live on testnet26 (solo):**
- With **no `algo=`** in `digibyte.conf`, run dgbminer with `-a sha256d`, then `scrypt`, `skein`, `qubit`
  and `odo`. Each must get its own algorithm's template (the node's `getblock` shows the matching
  `pow_algo`), and blocks must be accepted.
- On a block found after the change, `getblock <hash> 2` must show the coinbase with the oracle
  commitment output (`6a…`) before the witness commitment, and the node must have accepted it.
- Check with a long `--coinbase-sig` (the buffer case) that the coinbase is built correctly, or that the
  signature is skipped with the existing warning, and never overruns.

## 7. Files

| File | Change |
|---|---|
| `algo/odo/odo.cpp` | interval from `opt_testnet` |
| `cpu-miner.c` | `--testnet` option; runtime GBT and long-poll requests with the algorithm and `digidollar-oracle`; oracle output; output count; compact-size script length; coinbase buffer sizing |
| `miner.h` | `extern bool opt_testnet;` and the usage text |
| `README.md` | `--testnet` for testnet26 Odo; `algo=` in `digibyte.conf` now optional; DigiDollar note |
| `design-documents/…` | this plan |

## 8. Implementation notes

- **`--testnet`** (option code 1070): `opt_testnet` in `cpu-miner.c`, declared in `miner.h`, with a usage
  line and README examples. `scanhash_odo` reads it on every call. `register_odo_algo` logs the interval
  in use, and it runs after option parsing, so the log is always accurate.
- **dgbminer's Odo crypto files are identical to DigiByte's** (`odocrypt.cpp`, `odocrypt.h`,
  `KeccakP-800-reference.cpp`); only comments and blank lines differ. So the hash itself is the one
  already verified against the node's `pow_hash` on five real blocks during the GSS work. With the
  10-day interval, the four mainnet vectors match. The only behavioural change in the Odo path is the
  interval.
- **Template requests:** `build_gbt_requests()` runs once after `register_algo_gate()` and replaces
  `gbt_req` and `gbt_lp_req` for DigiByte's five algorithms only. The long-poll string is still a
  format string (`%%s` becomes `%s` for the long-poll id).
- **Oracle output:** parsed from `default_oracle_commitment`. A full `6a…` script is used verbatim;
  bare data is wrapped in `OP_RETURN` with a push (up to 255 bytes, otherwise skipped). A malformed or
  oversized commitment is skipped with a warning, because the block is still valid without it. The
  output count is `1 + oracle + segwit`, the length is compact-size (`varint_encode`), and the oracle
  output goes before the witness commitment.
- **Buffer:** `malloc(256 + 8 + 9 + oracle_script_size)`, which keeps the original 256-byte allowance
  for everything else, including the scriptSig extension.

## 9. Addition: Bech32m payout addresses and default coinbase text

**Bech32m (Taproot, `dgb1p…`).** `--coinbase-addr` rejected every witness v1+ address, for two reasons:
1. `bech32_decode` accepted only the Bech32 checksum constant (`chk == 1`). Bech32m (BIP350) needs
   `0x2bc830a3`. It now returns which encoding matched, and `segwit_addr_decode` enforces BIP350: v0
   must be Bech32, and v1 and later must be Bech32m.
2. **The payout-script buffer was too small.** `pk_script` was 26 bytes and the SegWit path was given
   `pk_buffer_size` (25) as its capacity. A 34-byte Taproot or **P2WSH** script didn't fit, so P2WSH
   was also broken. `pk_buffer_size` is the Base58 decode length and must stay 25, so the SegWit path
   now uses a separate `PK_SCRIPT_MAX` (42: a 40-byte witness program + 2). The coinbase buffer gained
   32 bytes of margin to match.

**Verified** with dgbminer's actual `util.c` address code (extracted unchanged) against vectors from a
BIP350 reference encoder, which itself reproduces the official BIP350 Taproot address exactly:

| Vector | New code | Original code |
|---|---|---|
| `dgb1p…` Taproot (Bech32m) | ✅ `5120…` | ❌ rejected |
| `dgbt1p…` Taproot, testnet | ✅ | ❌ rejected |
| `dgb1q…` P2WPKH (Bech32) | ✅ `0014…` | ✅ |
| `dgb1q…` P2WSH, 32-byte (Bech32) | ✅ `0020…` | ❌ rejected |
| BIP350 official `bc1p0xlx…` | ✅ | ❌ rejected |
| v1 with a Bech32 checksum (invalid) | ✅ rejected | ✅ rejected |
| v0 with a Bech32m checksum (invalid) | ✅ rejected | ✅ rejected |

**Coinbase text.** `coinbase_sig` now defaults to `GoSlimStratum`. It only applies when dgbminer
builds the coinbase itself (solo, `--no-stratum`). Override it with `--coinbase-sig=TEXT`, or turn it
off with `--coinbase-sig=""`. The existing 100-byte scriptSig limit still applies.

## 10. Live test results (2026-09-27, testnet26)

| Feature | Result |
|---|---|
| `--testnet` Odo key | ✅ Logs `Odo key interval: 1 day (testnet)`. Mined through GSS (stratum) normally. |
| Algorithm in template request | ✅ The `-P` dump shows `"rules": ["segwit", "digidollar-oracle"]}, "odo"]`, and the node returned an Odo template. |
| `digidollar-oracle` rule requested | ✅ In the same request. |
| Coinbase text | ✅ The submitted coinbase scriptSig is `03fec006` (height 442622) + `0d 476f536c696d5374726174756d` ("GoSlimStratum"). |
| Solo block | ✅ `submitblock` returned `null`: accepted (`BLOCK SOLVED 1`). |
| Bech32m (Taproot) payout | ✅ Confirmed working by Scott. |
| Oracle commitment output | ⏳ **Not exercised.** The node only returns `default_oracle_commitment` when an oracle bundle is ready (`AddOracleBundleToBlock` in `src/node/miner.cpp`). testnet26 had no bundles; GSS and other miners weren't getting any either. The template had no commitment, so dgbminer correctly built a 2-output coinbase (payout + witness). |

**Still to do:** run against a **mainnet** node, where DigiDollar is active, with `-D` and without `--testnet`.
Each template should log `GBT: DigiDollar oracle commitment included (N bytes)`. Planned after the
GSS 5.3.0 release.

**Possible follow-up:** DigiByte 9.26.6rc1 templates include an `odokey` field for Odo. Solo mode
could use it directly, which would remove the need for `--testnet` when mining solo.
