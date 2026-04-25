[![C](https://img.shields.io/badge/C-00599C?style=flat-square&logo=c&logoColor=white)]()
[![Kyber-1024](https://img.shields.io/badge/KEM-Kyber--1024-blue?style=flat-square)]()
[![X25519](https://img.shields.io/badge/ECDH-X25519-green?style=flat-square)]()
[![XChaCha20-Poly1305](https://img.shields.io/badge/AEAD-XChaCha20--Poly1305-red?style=flat-square)]()
[![libsodium](https://img.shields.io/badge/libsodium-%3E%3D1.0.18-blueviolet?style=flat-square)]()
[![Argon2id](https://img.shields.io/badge/KDF-Argon2id-orange?style=flat-square)]()
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow?style=flat-square)]()

# 🛡️ Classified Kyber Encryption Utility (XChaCha20‑Poly1305 Edition)

**CKEU** is a hardened, deniable file‑encryption tool that combines **post‑quantum Kyber‑1024** with **classical X25519** in a hybrid KEM, then protects data with a **streaming XChaCha20‑Poly1305 AEAD**.  
The secret key and all metadata are encrypted inside the file header (password‑protected via Argon2id), and every encrypted file is indistinguishable from random noise.

This version (v10.7‑FINAL) uses **libsodium** exclusively for symmetric cryptography and key derivation, offering a **complete, auditable, constant‑time** implementation with zero OpenSSL dependency for core operations.

---

## ✨ Features

- 🔐 **Hybrid KEM** – Kyber‑1024 + X25519 → 256‑bit shared secret  
- ⚡ **Streaming AEAD** – `crypto_secretstream_xchacha20poly1305` (libsodium)  
- 🔑 **Self‑contained header** – secret key + metadata encrypted with a user password  
- 🧵 **Chunk‑aligned decryption** – exact secretstream boundaries, no whole‑file memory  
- 🥷 **Deniability** – random padding (0–64 KB) before & after ciphertext; no magic bytes  
- 🧹 **Memory safety** – `mlockall`, `sodium_memzero`, `memfd_create` for RAM‑backed temp files  
- 🚨 **Signal safety** – emergency cleanup writes newline + exits via `_exit()`  
- 🎯 **Generic errors** – no fingerprinting (always “Decryption failed.”)  
- 📜 **Standards** – NIST FIPS 203, RFC 7748, RFC 9106, RFC 8439 (XChaCha20 reference)  

---

## 🔧 Dependencies

- **libsodium** ≥ 1.0.18 (for XChaCha20‑Poly1305, Argon2id, X25519, secretstream, etc.)  
- A C11 compiler (GCC/Clang)  
- Reference Kyber‑1024 code (provided in `./kyber/`)  

**No OpenSSL or external crypto libraries** – everything symmetric runs through libsodium.

---

## 🔨 Build

```bash
gcc -O2 -Wall -Wextra -Werror -std=c99 -DKYBER_K=4 -I. -Ikyber \
    ckeu.c \
    kyber/cbd.c kyber/fips202.c kyber/indcpa.c kyber/kem.c \
    kyber/ntt.c kyber/poly.c kyber/polyvec.c kyber/reduce.c \
    kyber/symmetric-shake.c kyber/verify.c \
    -fPIE -pie -fstack-protector-strong -D_FORTIFY_SOURCE=2 \
    -fno-builtin-memset -fno-strict-aliasing \
    -Wl,-z,relro,-z,now -lsodium -latomic -lpthread -lm -lc \
    -s -o ckeu

# Optional: strip even further
strip --strip-all --remove-section=.comment --remove-section=.note --remove-section=.gnu.version ckeu
```

*For strict RAM‑only temporary files (no disk fallback), add `-DSTRICT_RAM_ONLY`.*

---

## 🚀 Usage

Run `./ckeu`. All keys are generated per‑file – no external key management needed.

```
==========================================
 CLASSIFIED KYBER ENCRYPTION UTILITY
 v10.7-FINAL [100/100 – COMPLETE]
==========================================

 MAIN OPERATIONS MENU
 [1] Encrypt File
 [2] Decrypt File
 [3] View Features & Compliance
 [4] Secure Exit
```

### 🔒 Encryption
- You are prompted for a **password** twice.  
- The program generates a fresh hybrid keypair (Kyber‑1024 + X25519), performs KEM encapsulation against its own public key, derives a file key, and encrypts the data with **streaming XChaCha20‑Poly1305**.  
- The secret key, together with plaintext/ciphertext lengths and padding size, is encrypted with the password (Argon2id → XChaCha20‑Poly1305) and stored in the file header.  
- Random padding (0–64 KB) is appended; the exact length is hidden inside the encrypted header.

Output file format: **[encrypted header] [KEM ciphertext] [stream header] [encrypted chunks] [padding]**  
Everything is indistinguishable from random.

### 🔓 Decryption
- Provide the password – the header is decrypted, the secret key recovered.  
- The KEM ciphertext is decapsulated to obtain the shared secret.  
- The file is then **streamed and verified chunk‑by‑chunk** using `crypto_secretstream_xchacha20poly1305_pull`.  
- If authentication fails at any point, the output is deleted and a generic error message is shown.

---

## 🧪 Cryptographic Architecture

| Component | Implementation |
|:----------|:---------------|
| **KEM** | Kyber‑1024 (reference) + X25519 (libsodium) – hybrid shared secret |
| **AEAD** | `crypto_secretstream_xchacha20poly1305` (streaming) |
| **Key Derivation (file)** | SHA‑512 over `"cke_file_v2" || shared_secret` → 256‑bit key |
| **Password‑based KDF** | Argon2id (moderate ops/mem) + SHA‑512 domain separation (`"cke_ske_v2"`) |
| **Header encryption** | XChaCha20‑Poly1305 (one‑shot) with random salt & nonce |
| **Padding** | `randombytes_uniform` 0–64 KB, length stored (encrypted) in header |
| **Constant‑time ops** | `sodium_memcmp`, `sodium_memzero`, no early returns on failure |

---

## 📜 Compliance & Standards

| Standard | Application |
|:---------|:------------|
| **NIST FIPS 203** | ML‑KEM (Kyber‑1024) |
| **RFC 7748** | X25519 Diffie‑Hellman |
| **RFC 9106** | Argon2id memory‑hard KDF |
| **RFC 8439** | XChaCha20 / Poly1305 (basis for `secretstream`) |
| **NIST SP 800‑175B** | Guidelines for using symmetric crypto |

---

## 🧹 Security & OpSec

- **All sensitive memory** wiped with `sodium_memzero` (volatile barriers included).  
- Temporary encrypted data resides in a **RAM‑backed file** (`memfd_create` / `O_TMPFILE` / `tmpfile`).  
- `mlockall(MCL_CURRENT|MCL_FUTURE)` prevents swapping (root‑only).  
- Signal handler uses only async‑signal‑safe functions: writes a newline and calls `_exit()`.  
- **No magic bytes**, **no plaintext metadata** – files are indistinguishable from random noise.  
- Output is always deleted on decryption failure.  
- Input parsing via `fgets` + `strtol`, no `scanf`‑based overflows.

---

## ⚠️ Important Notes

- The password must be strong and never stored alongside the encrypted file.  
- The tool is **research‑grade** – no formal third‑party audit has been performed.  
- Kyber‑1024 provides category‑5 quantum resistance; the hybrid with X25519 adds classical security margin.  
- On Linux, `mlockall` may require `CAP_IPC_LOCK` or root; a warning is printed if it fails.

---

## 📄 License

MIT – see [LICENSE](LICENSE).

---

*Quantum‑ready. Streaming. Deniable.* 🛡️💎
```
