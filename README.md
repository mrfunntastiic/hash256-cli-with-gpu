# SUDAH ADA VERSI BARU OPTIMALISASI BISA KE https://github.com/mrfunntastiic/hash256-mine

# HASH Token CPU Miner (Rust)

Port Rust dari `miner.js` di folder induk. Native, multi-threaded, dan jauh lebih cepat dari versi Node.js.

## Kenapa Rust?

| Hal | Node.js (`miner.js`) | Rust (di sini) |
|---|---|---|
| Hashing | single-threaded JS | semua core via `std::thread::scope` |
| Per-attempt cost | `await contract.currentEpoch()` per nonce (RPC roundtrip!) | murni CPU, RPC-nya cuma di-poll tiap 15 detik |
| Hash rate (laptop 8-core) | ~1k–5k H/s | ~ratusan ribu – jutaan H/s |
| Binary | butuh Node + 200 MB `node_modules` | satu binary statis |

## Requirements

- Rust toolchain (>= 1.75) — install via [rustup.rs](https://rustup.rs)
- Wallet Ethereum dengan ETH untuk gas
- RPC endpoint (default `https://eth.llamarpc.com` — publik, ganti ke Alchemy/Infura punyamu kalau bisa)

## Build

```powershell
cd rust-miner
cargo build --release
```

Binary keluar di `target/release/hash-miner-rs.exe`.

## Run

### Cara 1: env var (paling aman)

PowerShell:
```powershell
$env:PRIVATE_KEY = "0xabc..."
$env:RPC_URL = "https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY"  # optional
$env:MINER_THREADS = "8"                                        # optional, default = num CPUs
cargo run --release
```

Bash:
```bash
PRIVATE_KEY=0xabc... cargo run --release
```

### Cara 2: prompt interaktif

```powershell
cargo run --release
# masukin private key pas diminta (input disembunyiin via rpassword)
```

## Konfigurasi

Ada di konstanta atas `src/main.rs`:

- `HASH_CONTRACT_ADDRESS` — `0xAC7b5d06fa1e77D08aea40d46cB7C5923A87A0cc`
- `DEFAULT_RPC_URL` — di-override pakai env `RPC_URL`
- `CHAIN_ID` — 1 (Ethereum Mainnet)
- `EPOCH_POLL_INTERVAL` — 15 detik (seberapa sering cek epoch berubah)
- `STATS_INTERVAL` — 2 detik (refresh hash rate display)

Gas tidak di-hardcode — alloy auto-estimate. Kalau mau force gas, edit bagian `contract.mint(...)` di `main.rs`.

## Cara kerja mining

Sama persis dengan versi JS:

1. **Challenge** = `keccak256(abi.encodePacked(chainId, contract, miner, epoch))`
2. **Proof** = `keccak256(abi.encodePacked(challenge, nonce)) < currentDifficulty`
3. **Reward** = `100 HASH >> (totalMints / 100_000)`

Bedanya di Rust:
- Tiap worker thread mulai dari `start_nonce + tid` dan inkremen pakai `stride = num_threads` — gak ada nonce yang dihash dua kali antar thread.
- `start_nonce` di-random tiap ronde supaya beberapa miner di wallet yang sama gak buang energi pada nonce yang sama.
- Watchdog `tokio` task ngecek epoch tiap 15 detik. Begitu epoch ganti, semua worker di-signal stop via `AtomicBool` dan ronde direstart.

## Stop

`Ctrl+C` — bakal signal worker buat berhenti setelah attempt sekarang, terus print statistik akhir.

## Security

- Jangan commit private key. Pakai env var atau prompt.
- RPC publik bisa rate-limit / di-MITM. Pakai punyamu kalau bisa.
- Mining butuh ETH untuk gas. Kalau hash rate ketemu solusi tapi gas-mu kurang, tx revert.

## License

MIT — risiko ditanggung sendiri.
