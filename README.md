# Dgbminer
A Digibyte optimized cpu miner. Sha256d, Scrypt, Skein, Qubit and Odocrypt.

# Build for Linux
```bash
sudo apt-get install build-essential binutils automake libssl-dev libcurl4-openssl-dev lib32z1-dev libjansson-dev libgmp-dev git

git clone https://github.com/Jongjan88/dgbminer/

cd dgbminer

chmod +x *.sh

./build.sh
```


***Run node && cpuminer on the same PC!***

## MMFP Solutions fork: DigiByte awareness

This fork ([mmfpsolutions/dgbminer](https://github.com/mmfpsolutions/dgbminer)) is maintained by
[MMFP Solutions](https://www.mmfpsolutions.io), makers of the GoSlimStratum solo mining pool, which mines
all five DigiByte algorithms including Odo. It adds three DigiByte fixes:

- **Correct Odo key on every network.** The Odocrypt key changes every **10 days on mainnet and
  regtest** and every **1 day on testnet**. The original hard-coded 1 day, so Odo mining was broken on
  mainnet 9 days out of 10. The default is now 10 days. **On testnet, add `--testnet`**, for both solo
  and pool (stratum) mining. The startup log shows which interval is in use.
- **Algorithm-aware solo mining.** dgbminer asks the node for a template for the algorithm you chose
  with `-a`. One node can serve all five algorithms, so `algo=` in `digibyte.conf` is no longer needed.
- **DigiDollar-aware solo mining.** dgbminer requests the `digidollar-oracle` rule and adds the oracle
  commitment to the coinbase when the node provides one.
- **Taproot and P2WSH payout addresses.** `--coinbase-addr` now accepts Bech32m addresses
  (`dgb1p…`, Taproot) and 32-byte SegWit v0 addresses (P2WSH). Before, the original accepted only
  Base58 and 20-byte `dgb1q…` addresses.
- **Coinbase text.** Solo-mined blocks carry `GoSlimStratum` in the coinbase by default. Change it
  with `--coinbase-sig=TEXT`, or turn it off with `--coinbase-sig=""`. Pool mining isn't affected,
  because there the pool builds the coinbase.

# Linux Testnet Solo - odo (note --testnet)
./cpuminer -a odo --testnet -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgbt1q... -D

# Linux Solo - sha256d
./cpuminer -a sha256d -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - scrypt
./cpuminer -a scrypt -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - skein
./cpuminer -a skein -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - qubit
./cpuminer -a qubit -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - odo
./cpuminer -a odo -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D



# Help
./cpuminer --help

# Missing libraries?
libgmp3-dev zlib1g-dev

# Mainnet config ( digibyte.conf )
`algo=` below is optional with this fork. dgbminer asks for the algorithm you pass with `-a`.
```bash
maxconnections=300
listen=1
server=1
algo=sha256d
#algo=scrypt
#algo=skein
#algo=qubit
#algo=odo

rpcuser=user
rpcpassword=pass
rpcallowip=0.0.0.0/0
rpcport=14022
```
# Testnet config ( digibyte.conf )
```bash
maxconnections=300
testnet=1
listen=1
server=1
algo=sha256d
#algo=scrypt
#algo=skein
#algo=qubit
#algo=odo

[test]
rpcuser=user
rpcpassword=pass
rpcallowip=0.0.0.0/0
rpcport=14022
```
