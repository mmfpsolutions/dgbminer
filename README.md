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

# Odocrypt on testnet: use --testnet
The Odocrypt key changes every 10 days on mainnet and regtest, and every day on testnet. The miner
uses the 10-day interval by default; add `--testnet` when mining odo on testnet (solo or pool). The
interval in use is shown at startup.

# Solo mining notes
- The miner asks the node for a block template for the algorithm given with `-a`, so a single node
  can serve every algorithm and `algo=` in `digibyte.conf` is optional.
- DigiDollar: the miner requests the `digidollar-oracle` template rule and includes the node's
  oracle commitment in the coinbase whenever one is provided.
- `--coinbase-addr` accepts Base58, Bech32 (`dgb1q...`, including P2WSH) and Bech32m (`dgb1p...`,
  Taproot) addresses.

# Linux Solo - sha256d
./cpuminer -a sha256d -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - scrypt
./cpuminer -a scrypt -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - skein
./cpuminer -a skein -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - qubit
./cpuminer -a qubit -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D

# Linux Solo - odo (add --testnet on testnet)
./cpuminer -a odo -o http://127.0.0.1:14022/ --userpass=user:pass --no-getwork --no-stratum --coinbase-addr=dgb1q66lmtmlkswlphp5j7fgvg4nar4y8uf24hvlu89 -D



# Help
./cpuminer --help

# Missing libraries?
libgmp3-dev zlib1g-dev

# Mainnet config ( digibyte.conf )
`algo=` is optional: the miner requests the algorithm given with `-a`.
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
