# STM32H750VBT Secure Boot Platform Evaluation

**Document ID:** PRD-EVAL-001
**Version:** 1.0
**Status:** Draft
**Last Review:** 2026-04-28
**Owner:** Platform Evaluation Team

---

## 1. Purpose

This document evaluates the STM32H750VBT microcontroller + BY25Q32ES external NOR flash as the Tier-1 reference implementation platform for the SBOP secure bootloader. It maps platform hardware capabilities to SBOP requirements, defines the flash layout, and identifies gaps requiring mitigation.

---

## 2. Platform Identification

### 2.1 Microcontroller

| Parameter | Value |
|---|---|
| **Part Number** | STM32H750VBT |
| **Package** | LQFP100 (14×14 mm) |
| **Core** | Arm Cortex-M7 @ 400 MHz (operational), double-precision FPU. 480 MHz max. |
| **HSE** | 25 MHz external oscillator |
| **Internal Flash** | 128 KB. I-Cache enabled (16 KB) for zero-wait instruction fetch from flash. |
| **Internal RAM** | 1 MB total (192 KB TCM + 512 KB AXI-SRAM + 352 KB SRAM1/2/3 + 4 KB backup) |
| **L1 Cache** | 16 KB I-Cache (enabled) + 16 KB D-Cache (disabled). I-Cache mandatory for internal flash execution. |
| **Supply Voltage** | 1.62–3.6 V |
| **Temperature** | −40 to +85 °C (industrial) |
| **Crypto Hardware** | AES-128/192/256 (ECB/CBC/GCM/CCM/CTR), HASH (SHA-1, SHA-256), HMAC, TRNG |
| **Unique ID** | 96-bit factory-programmed |
| **Debug** | SWD + JTAG (4 KB embedded trace buffer) |
| **Security** | ROP (Level 1/2), PC-ROP, Secure Access Mode, 3× active tamper pins |
| **USART** | USART1: PA9 (TX) / PA10 (RX), 96 MHz APB2 clock |
| **SPI Interface** | 6× SPI (SPI1–SPI6), plus dual Quad-SPI @ 133 MHz. SPI1 on APB2. |
| **GPIO** | All SPI1 pins configured Medium speed (OSPEEDR = 0b01) |
| **Reference Manual** | RM0433 Rev 8 (January 2023) |
| **Datasheet** | DS12556 Rev 8 (January 2026) |

### 2.2 External NOR Flash

| Parameter | Value |
|---|---|
| **Part Number** | BY25Q32ES |
| **Density** | 32 M-bit (4 MB) |
| **Interface** | Standard SPI Mode — SCLK, /CS, SI (IO0), SO (IO1), /WP (IO2), /HOLD (IO3) |
| **Clock Rate** | 120 MHz maximum (Fast Read) |
| **Sector Size** | 4 KB uniform (1024 sectors) |
| **Block Sizes** | 32 KB / 64 KB |
| **Security Registers** | 3× 1024-byte with independent OTP locks (LB1, LB2, LB3) |
| **Unique ID** | 128-bit factory-programmed (4BH command) |
| **Write Protection** | /WP hardware pin + software BP0–BP4 block protect + SRP0/SRP1 status register protect |
| **Endurance** | 100K program-erase cycles per sector (typical) |
| **Retention** | 20 years data retention (typical) |
| **Supply** | 2.7–3.6 V |
| **Temperature** | −40 to +85 °C (industrial) |
| **Datasheet** | QA-DT-006 Rev 2.3 |

### 2.3 Interface Configuration

```
STM32H750VBT (SPI1, Standard Mode, 80 MHz)    BY25Q32ES
┌──────────────────────────────┐            ┌──────────────┐
│  PA5  ── SCLK ──────────────>│────────────│ SCLK          │   (AF5, Medium speed)
│  PA6  ── MISO <──────────────│────────────│ SO (IO1)      │   (AF5, Medium speed)
│  PA7  ── MOSI ──────────────>│────────────│ SI (IO0)      │   (AF5, Medium speed)
│  PA4  ── NSS  ──────────────>│────────────│ /CS           │   (AF5, Medium speed)
└──────────────────────────────┘            │ /WP (IO2)     │   (external resistor, pull high)
                                            │ /HOLD (IO3)   │   (external resistor, pull high)
                                            └──────────────┘

USART1 (Debug / Recovery Console):
  PA9  ── TX   (AF7, Low speed)
  PA10 ── RX   (AF7, Low speed)
```

**Note:** Standard SPI mode selected. Quad SPI not required for bootloader operation. SPI1 pins configured at Medium speed; USART1 pins at Low speed (OSPEEDR = 0b00) — Low speed supports up to ~10 Mbps, sufficient for 3 Mbps UART with margin.

### 2.4 Clock Configuration

```
HSE (25 MHz external oscillator)
  │
  ├── PLL1: 25 MHz / 5 × 200 / 2 = 500 MHz (PLL1R)
  │   ├── sys_ck = 400 MHz  (PLL1R / DIVP1 = 500 / 1.25? no)
  │   │   Actually: 25 / 5 * 160 / 2 = 400 MHz
  │   │   PLL1: DIVM=5 (25/5=5 MHz), DIVN=160 (5*160=800 MHz), DIVP=2 (800/2=400 MHz)
  │   └── CPU @ 400 MHz (HCLK = sys_ck, APB1/2 prescaled)
  │
  ├── PLL2: 25 MHz / 5 × 160 / 2 = 400 MHz
  │   ├── SPI1 clock source = PLL2R (via SPI1SEL)
  │   ├── SPI1 kernel clock = 160 MHz (PLL2Q = DIVR × DIVQ path)
  │   │   Configured: PLL2 DIVM=5, DIVN=160, DIVP=2, DIVQ=2.5
  │   │   → PLL2R = 400 MHz (for sys_ck backup or peripheral)
  │   │   → SPI1 kernel = PLL2Q configured for 160 MHz
  │   │   → SPI1 prescaler = DIV2 → SCLK = 80 MHz
  │   └── USART1 clock source = PLL2Q (96 MHz via USART1SEL)
  │
  └── HSI (64 MHz internal) — fallback during clock fail
```

**Clock verification table:**

| Clock | Source | Target | Divider Path | Actual | Tolerance |
|---|---|---|---|---|---|
| **HCLK** (CPU) | HSE → PLL1R | 400 MHz | DIVM=5, DIVN=160, DIVP=2 | 400.0 MHz | ±0% |
| **SPI1 kernel** | HSE → PLL2Q | 160 MHz | DIVM=5, DIVN=160, DIVQ=2.5 | 160.0 MHz | ±0% |
| **SPI1 SCLK** | PLL2Q | 80 MHz | SPI prescaler DIV2 | 80.0 MHz | ±0% |
| **USART1** | PLL2Q | 96 MHz | USART1SEL = PLL2Q | 96.0 MHz | ±0% |
| **APB1** | HCLK | 200 MHz | DIV2 (max 200 MHz) | 200.0 MHz | ±0% |
| **APB2** | HCLK | 200 MHz | DIV2 (max 200 MHz) | 200.0 MHz | ±0% |
| **HASH / CRYP** | HCLK | 400 MHz | Direct from HCLK | 400.0 MHz | ±0% |

**SPI1 timing verification (80 MHz):**

```
SPI1 kernel clock: 160 MHz
SPI1 prescaler:    DIV2 → SCLK = 80 MHz
SPI1 period:       12.5 ns

BY25Q32ES requirements at 80 MHz:
  - SCLK period:  ≥ 12.5 ns  ✓ (8.3 ns at 120 MHz max, 80 MHz is conservative)
  - tCH / tCL:    ≥ 4.5 ns   ✓ (6.25 ns each at 50% duty cycle)
  - tCSS:         ≥ 5 ns     ✓ (CS setup to first SCLK)
  - tCSH:         ≥ 5 ns     ✓ (last SCLK to CS high)
  - tSU:DATA:     ≥ 2 ns     ✓ (data setup before SCLK rising edge)
  - tHD:DATA:     ≥ 3 ns     ✓ (data hold after SCLK rising edge)

80 MHz SPI effective throughput: ~10 MB/s (with Fast Read 0x0B command overhead).
512 KB payload read: ~51 ms (was ~35 ms at 120 MHz, still within budget).
```

**I-Cache verification:**

```
Internal Flash latency at 400 MHz HCLK:
  - Flash access time: ~2.5 ns per access (with ART accelerator)
  - Without I-Cache:  5–7 wait states = 12.5–17.5 ns per instruction fetch
  - With I-Cache:     zero wait states on cache hit (~95% hit rate for bootloader loop)
  - I-Cache enabled at INIT phase (Phase 1) before any SPI communication
  - D-Cache disabled — bootloader stack and data reside in DTCM (zero-wait, deterministic)
  - DTCM eliminates need for D-Cache; DMA not used by bootloader, so no buffer coherency concerns
```

---

## 3. Architecture Decisions

### 3.1 Decision: No MPU / Single-Threaded Bootloader

**Decision:** The Cortex-M7 Memory Protection Unit (MPU) is **not used** for Zone-1/Zone-2 isolation.

**Rationale:**

The SBOP bootloader on this platform operates as a **strictly single-threaded, run-to-completion** state machine. There are never two concurrent execution contexts (no RTOS, no interrupts during critical verification phases, no background tasks). The boot flow is:

```
RESET → INIT → SELECT_SLOT → LOAD_IMAGE → PARSE_HEADER →
VERIFY_SIGNATURE → VERIFY_INTEGRITY → CHECK_VERSION →
COMMIT_VERSION → MARK_ACTIVE → LOCK_BOOT → EXECUTE (jump to app)
```

Once the `EXECUTE` phase transfers control to the application firmware in Zone 2, the bootloader is **terminated** — it does not co-reside with the application. The application runs in a separate address space (external NOR flash) and has its own stack, vector table, and runtime.

**Zone-1 memory protection is instead enforced by:**

| Mechanism | Protection |
|---|---|
| **PC-ROP** (Proprietary Code Read-Out Protection) | Internal flash sectors containing bootloader .text and .rodata are marked execute-only. Any read access from code outside the protected sectors returns zero. This includes the KD key handle and embedded public keys. |
| **ROP Level 1** (Read-Out Protection) | JTAG/SWD memory read is disabled. Flash cannot be read via debugger. Only mass-erase or sector-erase of non-protected sectors is permitted. |
| **Secure Access Mode** (optional) | Bootloader can be placed in secure user flash areas. After secure service execution completes, the secure areas are automatically locked until next reset. |
| **Single-thread guarantee** | No DMA access to internal flash during boot. All interrupts disabled during verification phases. SPI1 polled (no DMA) to avoid concurrency. |
| **Boot complete → jump away** | After LOCK_BOOT, the bootloader explicitly clears sensitive state from RAM (KD, derived keys, verification results), sets the entry point, and executes a tail-call jump to the application. There is no bootloader code left resident or callable from Zone 2. |

**Implication:** Zone isolation is achieved by **temporal separation** (boot → lock → jump → never return) combined with **hardware access control** (PC-ROP prevents reads from non-boot code). The MPU is an unnecessary complexity for a single-context bootloader.

---

### 3.2 Decision: NOR Flash Security Registers as Trust Anchor

**Decision:** The three OTP-lockable Security Registers in the BY25Q32ES serve as the **hardware root of trust storage** for the bootloader. The BY25Q32ES provides 3× 1024-byte SRs, but **only the first 256 bytes of each SR are used** for cross-vendor compatibility (e.g., Winbond W25Q32JV JEDEC ID 0xEF 0x40 0x16 provides 3× 256-byte SRs). All field offsets are within the 0x00–0xFF range.

**Register Allocation:**

| Register | Address Range | Max Usable | OTP Lock Bit | Contents | Size | Access After Lock |
|---|---|---|---|---|---|---|
| **SR1** | 00_1000h – 00_10FFh | 256 B | LB1 (SR2[13]) | Version counter (32-bit × 4 redundant copies), boot slot selector, boot flags, min_allowed_version | ~64 bytes used | Read-only |
| **SR2** | 00_2000h – 00_20FFh | 256 B | LB2 (SR2[12]) | KD commitment: HMAC-SHA-256(KD, "SBOP-DEVICE-COMMIT") — 32 bytes. Proves possession of KD without storing it raw. | 32 bytes used | Read-only |
| **SR3** | 00_3000h – 00_30FFh | 256 B | LB3 (SR2[11]) | NOR Flash UID binding: HMAC-SHA-256(NOR_UID, MCU_UID) — 32 bytes. Binds this specific NOR flash chip to this specific MCU. | 32 bytes used | Read-only |

**Lock sequence during provisioning:**

```
1. MCU UID read (96-bit), NOR UID read (128-bit, 4BH command)
2. KD = HKDF-SHA-256(KR, MCU_UID || NOR_UID) — derived at provisioning station HSM
3. KD commitment computed: HMAC-SHA-256(KD, "SBOP-DEVICE-COMMIT")
4. NOR-MCU binding computed: HMAC-SHA-256(NOR_UID, MCU_UID)
5. Write version counter (initial = 1) to SR1
6. Write KD commitment to SR2
7. Write NOR-MCU binding to SR3
8. Set LB1, LB2, LB3 = 1 (OTP permanent lock)
9. Verify SR1, SR2, SR3 read-back matches written values
10. Set ROP Level 1 on MCU
```

**After lock:** The security registers can never be modified. The KD commitment proves the device was provisioned with the correct KD without exposing KD itself. The NOR-MCU binding prevents flash transplantation attacks.

---

### 3.3 Decision: NOR Flash UID for Anti-Cloning

**Decision:** The 128-bit factory-programmed NOR Flash Unique ID (4BH command) is combined with the 96-bit MCU Unique ID to create a **224-bit composite device identity** that is cryptographically bound during provisioning.

**Anti-cloning chain:**

```
NOR Flash UID (128-bit, read-only, factory-set)
        +
MCU UID (96-bit, read-only, factory-set)
        ↓
    Composite UID = MCU_UID || NOR_UID   (224 bits)
        ↓
    KD = HKDF-SHA-256(KR, Composite UID)  ← Derived at provisioning station
        ↓
    Binding = HMAC-SHA-256(NOR_UID, MCU_UID)  ← Stored in SR3 (OTP)
        ↓
    Commitment = HMAC-SHA-256(KD, "SBOP-DEVICE-COMMIT")  ← Stored in SR2 (OTP)
```

**Clone detection at boot time:**

```
1. Read MCU UID (96-bit from STM32 Unique ID registers, 0x1FF1_E800)
2. Read NOR UID (128-bit via 4BH SPI command)
3. Read binding from SR3
4. Verify: HMAC-SHA-256(NOR_UID, MCU_UID) == binding_from_SR3
   → If mismatch: ERR-ID-CLONE-001 → FAILSAFE
5. Read KD commitment from SR2
6. Derive KD = HKDF-SHA-256(KR, MCU_UID || NOR_UID)  ← KR from HSM at provisioning station, KD injected
7. Verify: HMAC-SHA-256(KD, "SBOP-DEVICE-COMMIT") == commitment_from_SR2
   → If mismatch: ERR-ID-IDENT-002 → FAILSAFE
```

**Protection against:**

| Attack | Defense |
|---|---|
| Transplant NOR flash to different MCU | NOR-MCU binding (SR3) mismatch → FAILSAFE |
| Clone NOR flash contents | NOR UID is factory-set, non-reproducible. Each NOR chip has a unique 128-bit ID. |
| Replace MCU with different chip | MCU UID differs → binding mismatch + KD derivation incorrect |
| Replay provisioning data | KD commitment (SR2) binds to specific composite UID |
| Backend registration duplicate | Backend stores composite UID; rejects duplicate registration |

---

### 3.4 Decision: X25519 Hybrid Encryption with Dual-Gate Verification

**Decision:** Firmware images are encrypted with a hybrid X25519 + AES-256-CTR scheme using four OTA key pairs (OTA_KEY_PAIR[0..3]). The backend holds `OTA_public[i]` (X25519 point) and encrypts firmware **once** per release — all devices download the same ciphertext. The bootloader holds `OTA_private[i]` (X25519 scalar) in PC-ROP .rodata for ECDH decryption. OTP stores SHA-256(OTA_public[i]) fingerprints for key verification plus a revocation bitmap. Extracting the device-side key (`OTA_private`) allows decryption but NOT encryption — an attacker cannot create valid encrypted firmware without the backend's `OTA_public` and the Ed25519 signing key.

**Key inventory:**

```
KR              = Root key (backend HSM, never leaves)
KD              = HKDF-SHA-256(KR, MCU_UID || NOR_UID)     ← Device key (unique per device)

OTA_KEY_PAIR[0..3]:
  OTA_private[i]  = X25519 scalar (32 B)                   ← Bootloader .rodata (PC-ROP Zone 1)
  OTA_public[i]   = OTA_private[i] * G (32 B)              ← Backend HSM (for ECDH encryption)
  SHA-256(OTA_public[i]) in SR1 OTP[192..319]              ← Fingerprint (verification)
  OTA revocation bitmap in SR1 OTP (one-way clear)          ← Revocation

KD_Auth         = HKDF-SHA-256(KD, "SBOP-OTA-AUTH")        ← OTA authentication key (32 B)
```

OTA_KEY_PAIR are **not per-device** — all devices share the same four key pairs. Per-device uniqueness is provided by the bootloader's KD re-encrypt on first boot (Stage 2). Extracting `OTA_private[i]` from a device enables decryption only; the Ed25519 firmware signature and backend-held `OTA_public[i]` prevent an attacker from creating and delivering malicious encrypted firmware.

**Two-stage encryption model:**

```
Stage 1 — OTA delivery (X25519 ECDH encrypted, FLAG_OTA_PENDING set):
  Backend: ephemeral e → E=e*G → S=X25519(e, OTA_public[i]) → K_s=HKDF(S)
           AES-256-CTR(K_s, IV, Payload) → all devices download same ciphertext.
  OTA library NEVER decrypts. NOR stores ciphertext only.
  Plaintext Payload exists NOWHERE in non-volatile storage.

Stage 2 — First boot after OTA (bootloader Gate 2):
  Bootloader: S = X25519(OTA_private[i], E) → K_s = HKDF(S)
  Decrypts with K_s → AXI SRAM (volatile).
  Verifies OTA key fingerprint + revocation (SR2 OTP).
  Verifies Ed25519 signature + hash in AXI SRAM.
  Re-encrypts with KD (per-device key) → writes back to NOR.
  Clears FLAG_OTA_PENDING.
  From this point, image is KD-encrypted at rest (per-device unique).

Stage 3 — All subsequent boots:
  Bootloader decrypts directly with KD → AXI SRAM.
  No ECDH needed. No conditional logic.
```

**Full encryption flow:**

```
Backend (once per firmware version, all devices share same ciphertext):
  Input:  Payload (plaintext application binary, ≤ 512 KB)
          OTA_public[key_index] (X25519 point, HSM-resident)
          KI_private[sig_key_index] (Ed25519, HSM-resident)

  e            = CSPRNG(32 bytes)                             ← Ephemeral X25519 scalar
  E            = e * G                                        ← Ephemeral public key (32 B, sent to device)
  S            = X25519(e, OTA_public[key_index])              ← ECDH shared secret
  K_s          = HKDF-SHA-256(S, "SBOP-OTA-ENC")              ← AES session key (32 B)
  iv           = CSPRNG(16 bytes)                             ← Random IV
  encrypted    = AES-256-CTR(K_s, iv, Payload)                ← Encrypt Payload only

  OTA wire package = ImageHeader          (80 B, plaintext)
                   || key_index           (1 B, in header.reserved[0])
                   || E                   (32 B, plaintext — ephemeral public key)
                   || iv                  (16 B, plaintext)
                   || encrypted           (N B, ciphertext)
                   || InnerSignatureBlock (116 B, plaintext — outside encryption envelope)

  InnerSignatureBlock is OUTSIDE the encryption envelope.
  Gate 1 can verify manifest signature + encrypted blob hash without any keys.
  Only Payload is encrypted.

OTA Library (Zone 2, during download):
  // NO DECRYPTION. NO ECDH. NO PLAINTEXT IN NOR.
  // OTA treats everything as opaque bytes except ImageHeader metadata.
  download ImageHeader (80 B)          → verify magic + version + board_id
  download E (32 B)                    → store as-is (plaintext ephemeral key)
  download iv (16 B)                   → store as-is
  download encrypted (N B)             → store as-is (CIPHERTEXT to NOR)
  download InnerSignatureBlock (116 B) → store as-is
  // Verify: SHA-256(E || iv || encrypted || InnerSigBlock) == manifest.encrypted_payload_hash
  // Verify: Manifest signature using embedded public key from ManifestSignatureBlock
  storage_set_slot_status(target_slot, VERIFIED)

Bootloader (Zone 1, Phase 5 — first boot after OTA):
  // Image has FLAG_OTA_PENDING set
  enc_key_index = header.reserved[0]

  // OTA key fingerprint + revocation check (before ECDH)
  Verify SHA-256(OTA_public[enc_key_index]) == SR2_OTP.ota_fingerprint[enc_key_index]
  Check SR2_OTP.ota_revocation >> enc_key_index & 1 == 1    // Key not revoked

  // ECDH + decrypt
  E  = image_data[80..112]                                   // Ephemeral public key (32 B)
  S  = X25519(OTA_private[enc_key_index], E)                 // ECDH shared secret
  K_s = HKDF-SHA-256(S, "SBOP-OTA-ENC")                     // AES session key
  Payload = AES-256-CTR_decrypt(K_s, iv, encrypted)          // → AXI SRAM (volatile only)
  secure_zeroize(S, 32); secure_zeroize(K_s, 32)             // Session secret no longer needed

  // Verify signature + hash in AXI SRAM
  Verify Ed25519 signature using embedded public key
  Verify SHA-256(payload) == header.hash

  // Re-encrypt with KD for per-device at-rest encryption
  Re-encrypt with KD → write back to NOR
  Clear FLAG_OTA_PENDING

Bootloader (Zone 1, Phase 5 — all subsequent boots):
  // FLAG_OTA_PENDING is clear → encrypted with KD
  Decrypt with KD → AXI SRAM (volatile only)
  Verify signing key fingerprint + revocation (SR1 OTP)
  Verify Ed25519 signature using embedded public key
  Verify SHA-256(payload) == header.hash
  Proceed to version check → EXECUTE
```

**Why the OTA library never decrypts:**

| Factor | OTA Decrypts (old) | OTA Never Decrypts (new) |
|---|---|---|
| Plaintext in NOR | Yes — from OTA completion until next boot | **Never** — only ciphertext in NOR |
| Power-off attack window | OTA completion → next boot | **None** — plaintext only in volatile AXI SRAM during ~300 ms boot |
| Key exposure in Zone 2 | OTA_private must be available in application context | OTA_private never leaves Zone 1 (bootloader .rodata, PC-ROP) |
| Code complexity | ECDH + decrypt + NOR write | Write opaque bytes + hash verify |
| Resume support | Must re-derive K_s and re-decrypt from start | HTTP Range resume on ciphertext — no key needed |

**Per-device uniqueness:**

| Property | Mechanism |
|---|---|
| Each device has unique KD | KD = HKDF(KR, composite UID). Composite UID is unique per device. |
| OTA image encrypted once for all devices | Backend ECDH-encrypts once with fixed OTA_public[i] — same ciphertext for all devices. |
| At-rest image is per-device unique | Bootloader re-encrypts with KD on first boot. Each device writes different IV + ciphertext. |
| Image at rest cannot be decrypted by other devices | Different KD → different decryption key → ciphertext renders as garbage on other devices. |
| OTA_private never exposed to application | OTA_private resides in bootloader .rodata (PC-ROP Zone 1). OTA library never accesses it. |
| Backend never exposes KD | KD stays in HSM; only the fixed OTA_public is used for ECDH encryption. |

**Performance:**

| Operation | Hardware | Time |
|---|---|---|
| X25519 ECDH (device, first boot only) | Software (µNaCl / hacl-star on Cortex-M7 @ 400 MHz) | ~15 ms |
| HKDF-SHA-256 derive K_s | Hardware HASH + HMAC | < 1 ms |
| AES-256-CTR decrypt (512 KB) | Hardware CRYP AES @ 80 MHz | ~27 ms |
| Re-encrypt with KD (512 KB) | Hardware CRYP AES | ~27 ms |
| **Total first-boot overhead** | | **~70 ms** (within 300 ms boot budget) |
| All subsequent boots (KD decrypt) | Hardware CRYP AES | ~27 ms |

**Why AES-256-CTR for the bulk cipher:**
- STM32H750 hardware AES accelerator supports CTR mode (CRYP AES block)
- CTR requires no padding (image is arbitrary length)
- Encrypt and decrypt are the same operation (XOR with keystream)
- Random IV per release prevents identical plaintext from producing identical ciphertext
- No authentication needed at encryption layer — Ed25519 signature over plaintext provides integrity

**Protection against:**

| Attack | Defense |
|---|---|
| Eavesdrop OTA traffic | Image is AES-256-CTR encrypted with ECDH-derived K_s. Even if TLS is compromised, ciphertext requires knowing OTA_private[i] to derive K_s. |
| Replay old OTA image | Random ephemeral e per release — replaying an old package produces wrong K_s after ECDH. Version check also prevents downgrade. |
| Extract image from NOR flash | Payload is ALWAYS ciphertext in NOR (AES-256-CTR with KD or K_s). Plaintext exists only in volatile AXI SRAM during the ~300 ms boot window. |
| Extract OTA_private from device | Enables decryption only. Cannot encrypt valid firmware (requires OTA_public[i] for ECDH + Ed25519 signing key for signature). |
| Power-off during OTA download | OTA_private is fixed in .rodata — always available. Partial download detected by hash mismatch in Gate 1. |
| Power-off during first boot (re-encrypt) | Write-before-erase: KD-encrypted data written to staging area (upper half of slot) before erasing original. If power fails, bootloader recovers from staging area on next boot. No data loss window. |
| OTA key compromise | 4-key rotation with OTP revocation. If OTA_private[i] is extracted, backend switches to OTA_public[j] and revokes key[i] via OTP bitmap clear. Blast radius limited to one firmware version. |

---

### 3.5 Decision: AXI SRAM Execution (512 KB Application Limit)

**Decision:** The bootloader copies the verified firmware image from external NOR flash into the AXI SRAM (512 KB) and executes it from there. The 1 MB slot size in NOR flash accommodates the image plus header (80 B) and signature block (116 B), with headroom for future growth.

**Rationale:**

```
STM32H750VBT RAM layout:
  0x2400_0000  AXI SRAM    512 KB   ← Application execution target
  0x2000_0000  DTCM        128 KB   ← Bootloader data (stack, .data, .bss, SPI buf, FIH state)
  0x3000_0000  SRAM1       128 KB   ← OTA download buffer (application runtime)
  0x3002_0000  SRAM2       128 KB   ← OTA work buffer (application runtime)
  0x3004_0000  SRAM3        96 KB   ← Available (application runtime)
  0x3800_0000  SRAM4        64 KB   ← Backup SRAM (tamper log, boot metrics)
  0x0000_0000  ITCM         64 KB   ← Critical ISR / FIH code (optional)
```

**Why AXI SRAM instead of execute-in-place from NOR:**

| Factor | AXI SRAM Execution | XIP from NOR Flash |
|---|---|---|
| **Performance** | AXI SRAM: zero wait states, 400 MHz | NOR SPI: 80 MHz operational (120 MHz max), ~4-cycle latency per read |
| **Execution speed** | Full 400 MHz, no cache misses on AXI | SPI reads are slow; I-Cache hit rate limited |
| **Security isolation** | RAM cleared before jump. No residual data in shared bus. | NOR bus physically accessible for probing during execution |
| **Size constraint** | 512 KB maximum application image | Up to 1 MB per slot |
| **Attack surface** | Bus between MCU and NOR only active during boot copy (~51 ms for 512 KB at 80 MHz) | Bus active continuously during execution — extended probing window |
| **Power** | Higher (512 KB copy energy) | Lower (no copy, but continuous SPI traffic) |

The 512 KB AXI SRAM limit is an **intentional constraint** for the first-stage bootloader. It enforces a compact, auditable firmware footprint and reduces the attack surface window — the SPI bus is only active for ~51 ms during the copy phase.

**Execution flow:**

```
Phase 11 (EXECUTE):
  1. Verify application image hash (already done in Phase 6)
  2. Copy payload from NOR slot to AXI SRAM base (0x2400_0000)
  3. Verify copy: SHA-256(AXI_SRAM[0..image_size]) == expected_hash
  4. Set vector table offset register (VTOR) = 0x2400_0000
  5. Clear all bootloader state from SRAM1/2/3 (zeroize KD, derived keys, SPI buffers)
  6. Set MSP = AXI_SRAM[0]  (initial stack pointer from vector table)
  7. Set PC = AXI_SRAM[4]   (reset handler from vector table)
  8. Branch to application — bootloader is terminated
```

**Slot size justification (1 MB each):**

```
Slot layout (1 MB = 0x10_0000 bytes):
  [0x00_0000]  ImageHeader       64 bytes     ← Magic, version, hash, flags, image_size
  [0x00_0040]  IV                16 bytes     ← Random AES-CTR IV (plaintext)
  [0x00_0050]  Firmware Payload   ≤ 512 KB    ← ALWAYS ciphertext in NOR (AES-256-CTR)
                                                ← Decrypted on-the-fly to AXI SRAM at boot
  [0x08_0050]  InnerSignatureBlock 116 bytes   ← Ed25519(KI_private[key_index]) over (ImageHeader||Payload)
  [0x08_00C4]  Padding / Reserved  ~512 KB    ← Headroom for larger images (future stages)
```

Current application ≤ 512 KB leaves ~512 KB headroom per slot. If a future platform revision increases AXI SRAM (e.g., STM32H755 has 1 MB), the same slot layout supports images up to ~1 MB without changes.

---

### 3.6 Decision: Fault Injection Hardening (FIH) from MCUboot

**Decision:** Software-level fault injection detection uses the **Fault Injection Hardening (FIH)** pattern from the MCUboot project, adapted for the STM32H750 platform.

**FIH principles applied to SBOP bootloader:**

```
// Critical check example: signature verification
// FIH pattern: no boolean early-return, use FIH_INT / FIH_DEC / FIH_RET

#define FIH_SUCCESS  0x0A5A5A5A   // Specific non-zero success value
#define FIH_FAILURE  0x0C3C3C3C   // Specific non-zero failure value
#define FIH_MASK     0x0F0F0F0F   // Detect single-bit flips

typedef volatile uint32_t fih_int;

static inline fih_int fih_eq(uint32_t a, uint32_t b) {
    // Constant-time comparison returning FIH_SUCCESS or FIH_FAILURE
    uint32_t diff = (a ^ b);
    // Force to all-1 or all-0 in constant time
    diff = (diff | (0 - diff)) >> 31;
    return diff ? FIH_FAILURE : FIH_SUCCESS;
}

static inline fih_int fih_dec(fih_int v) {
    // Decrement, saturating at minimum (resistant to underflow glitch)
    if ((v & FIH_MASK) != FIH_SUCCESS) return FIH_FAILURE;
    // Intentional: single decrement, redundant check after
    v--;
    return v;
}

static inline fih_int fih_ret(fih_int v) {
    // Convert FIH value to boolean: only FIH_SUCCESS returns true
    return (v & FIH_MASK) == FIH_SUCCESS;
}
```

**Critical verification path with FIH:**

```
function sbop_verify_image(image_data, header, sig_block) -> Result:
    fih_int fih_rc = FIH_SUCCESS;

    // 1. Magic check (FIH-protected)
    if header.magic != 0x53424F50:
        fih_rc = fih_dec(fih_rc);
    // FIH: No early return — continue through all checks

    // 2. Header version check
    if header.header_version > MAX_SUPPORTED_VERSION:
        fih_rc = fih_dec(fih_rc);

    // 3. Image size check
    if header.image_size == 0 or header.image_size > MAX_IMAGE_SIZE:
        fih_rc = fih_dec(fih_rc);

    // 4. Hash verification (redundant)
    computed_hash = sha256(payload);
    hash_match = constant_time_compare(computed_hash, header.hash, 32);
    if fih_eq(hash_match, true) != FIH_SUCCESS:
        fih_rc = fih_dec(fih_rc);

    // 5. Redundant hash check (glitch detection)
    // Re-read header from flash; recompute hash; compare again
    header2 = re_read_header_from_flash();
    computed_hash2 = sha256(payload);
    hash_match2 = constant_time_compare(computed_hash2, header2.hash, 32);
    if hash_match != hash_match2:
        handle_fault_injection_detected();  // FAILSAFE — no return

    // 6. Signature verification
    sig_valid = ed25519_verify(sig_data, sig_block.signature, public_key);
    if fih_eq(sig_valid, true) != FIH_SUCCESS:
        fih_rc = fih_dec(fih_rc);

    // 7. Redundant signature check
    sig_valid2 = ed25519_verify(sig_data, sig_block.signature, public_key);
    if sig_valid != sig_valid2:
        handle_fault_injection_detected();

    // 8. Version check (redundant)
    stored_version = read_version_counter();
    if header.firmware_version < stored_version:
        fih_rc = fih_dec(fih_rc);
    stored_version2 = read_version_counter();
    if stored_version != stored_version2:
        handle_fault_injection_detected();

    // 9. Final FIH return — only FIH_SUCCESS passes
    if fih_ret(fih_rc) != true:
        return FAILSAFE;

    return OK;
```

**FIH protections against specific glitch types:**

| Glitch | FIH Defense |
|---|---|
| Skip comparison instruction | No single branch determines result; all checks contribute to `fih_rc` accumulator |
| Flip comparison result | `fih_eq()` uses constant-time diff; single-bit flip produces FIH_FAILURE, not FIH_SUCCESS |
| Skip function call / NOP slide | Redundant checks (hash, signature, version all double-checked). Mismatch between redundant checks triggers tamper response. |
| Clock glitch — corrupt register | FIH_SUCCESS and FIH_FAILURE are multi-bit patterns (0x0A5A5A5A / 0x0C3C3C3C); random bit flips produce neither |
| Voltage glitch — corrupt flash read | Header re-read from flash; hash recomputed; version counter re-read. Mismatch detected. |
| EM glitch — skip entire check block | State machine tracks phase progression; skipping a phase triggers ERR-BOOT-STATE-002 |

**FIH_SUCCESS/FIH_FAILURE design:**

```
FIH_SUCCESS = 0xA5A5A5A5   (binary: 1010_0101_...)
FIH_FAILURE = 0xC3C3C3C3   (binary: 1100_0011_...)

Hamming distance: 16 bits between the two values.
Probability a random glitch flips FIH_FAILURE into FIH_SUCCESS: ~2^-16
```

**DTCM FIH State Isolation with Guard Regions:**

FIH state variables are a high-value target for localized EM/laser attacks. A single flipped bit in the accumulator can turn failure into success. To harden against this, FIH state is placed in a dedicated DTCM region with guard words and complementary storage.

```
// DTCM FIH guard region (placed in dedicated linker section .fih_state)
#define FIH_GUARD_MAGIC   0xF1F1F1F1   // Guard word sentinel

typedef struct {
    volatile uint32_t guard_pre;        // MUST be FIH_GUARD_MAGIC
    volatile fih_int   fih_rc;          // Primary accumulator
    volatile fih_int   fih_rc_shadow;   // Complementary shadow (~primary)
    volatile uint32_t guard_mid;        // MUST be FIH_GUARD_MAGIC
    volatile uint32_t redundancy_ok;    // 1 if redundant checks passed
    volatile uint32_t redundancy_shadow;// Complementary shadow
    volatile uint32_t guard_post;       // MUST be FIH_GUARD_MAGIC
} fih_state_t;

// Linker: place in dedicated 32-byte DTCM region at 0x2000_FF00
// Surrounding 32 bytes on each side are reserved as guard gap (never touched)
__attribute__((section(".fih_state"))) volatile fih_state_t fih_state;
```

**Guard verification macro:**

```
#define FIH_GUARD_CHECK()  do { \
    if (fih_state.guard_pre  != FIH_GUARD_MAGIC || \
        fih_state.guard_mid  != FIH_GUARD_MAGIC || \
        fih_state.guard_post != FIH_GUARD_MAGIC) { \
        /* Guard word corrupted — possible localized EM/laser attack */ \
        handle_fault_injection_detected(); \
    } \
    /* Verify complementary shadow consistency */ \
    if ((fih_state.fih_rc ^ fih_state.fih_rc_shadow) != 0xFFFFFFFF) { \
        handle_fault_injection_detected(); \
    } \
    if ((fih_state.redundancy_ok ^ fih_state.redundancy_shadow) != 0xFFFFFFFF) { \
        handle_fault_injection_detected(); \
    } \
} while(0)
```

**Integration into critical verification path:**

```
function sbop_verify_image(image_data, header, sig_block) -> Result:
    FIH_GUARD_CHECK();  // Verify guard integrity before starting

    fih_state.fih_rc = FIH_SUCCESS;
    fih_state.fih_rc_shadow = ~FIH_SUCCESS;  // Complementary

    // 1. Magic check (FIH-protected)
    FIH_GUARD_CHECK();  // Guard check at every decision point
    if header.magic != 0x53424F50:
        fih_state.fih_rc = fih_dec(fih_state.fih_rc);
        fih_state.fih_rc_shadow = ~fih_state.fih_rc;

    // ... (remaining checks follow same pattern) ...

    // 9. Final FIH return
    FIH_GUARD_CHECK();
    if fih_ret(fih_state.fih_rc) != true:
        return FAILSAFE;

    return OK;
```

**DTCM memory layout with FIH isolation:**

```
0x2000_0000  DTCM start
0x2000_0000  Bootloader .data / .bss              ~12 KB
0x2000_3000  Stack (4 KB)
0x2000_4000  SPI buffer (4 KB)
0x2000_5000  General workspace
...
0x2000_FEE0  Guard gap (32 B, never touched)       ← Detects overshoot attacks
0x2000_FF00  .fih_state (32 B, isolated region)    ← FIH guard words + state
0x2000_FF20  Guard gap (32 B, never touched)       ← Detects overshoot attacks
0x2000_FF40  Reserved
0x2001_0000  DTCM end (128 KB boundary)
```

**Attack coverage:**

| Attack | Defense |
|--------|---------|
| EM pulse targeting FIH accumulator | Complementary shadow — single pulse can't flip both `fih_rc` and `fih_rc_shadow` to produce valid (FIH_SUCCESS, ~FIH_SUCCESS) pair |
| Laser fault on DTCM bit | Guard words at 3 positions (pre, mid, post) detect localized bit flips |
| Overshoot EM attack | 32-byte guard gaps with known 0x00 pattern — any non-zero byte indicates attack |
| Register swap glitch | `fih_state` is `volatile` — compiler cannot optimize away re-reads |
| Power glitch during guard check | Guard check macro repeats at every decision point; single missed check doesn't bypass |

---

### 3.7 Decision: OTA as Application Service Library

**Decision:** OTA update logic (authentication, download, decryption, writing to inactive slot, activation) is **not part of the bootloader**. It is provided as a service library (`libsbop-ota`) linked into the application firmware. The bootloader is only responsible for secure boot verification and image execution.

**Rationale:**

The bootloader has one job at power-on: verify the selected slot's image is authentic and untampered, then execute it. OTA is an application-level concern — the device is already running, has network connectivity, and has the runtime resources (TLS stack, HTTP client, larger buffers) needed for a reliable update.

**Separation of concerns:**

```
┌─────────────────────────────────────────────────────┐
│ Zone 1: Bootloader (internal flash, 128 KB)         │
│                                                     │
│  INIT → SELECT_SLOT → LOAD_IMAGE → PARSE_HEADER    │
│  → VERIFY_SIGNATURE → VERIFY_INTEGRITY              │
│  → CHECK_VERSION → COMMIT_VERSION → MARK_ACTIVE     │
│  → LOCK_BOOT → COPY_TO_AXI → EXECUTE                │
│                                                     │
│  Bootloader never:                                  │
│    - Connects to network                            │
│    - Authenticates with backend                     │
│    - Downloads firmware images                      │
│    - Writes to inactive slot                        │
│    - Handles HTTP/TLS                               │
│                                                     │
│  Bootloader only:                                   │
│    - Reads ciphertext from NOR (selected slot)     │
│    - ECDH with OTA_private + E → K_s / decrypt with KD → AXI SRAM (volatile) │
│    - Verifies key fingerprint + revocation (OTP)   │
│    - Verifies Ed25519 signature (embedded key)     │
│    - Verifies SHA-256 hash                          │
│    - Re-encrypts with KD → writes back to NOR       │
│    - Checks version anti-rollback                   │
│    - Two-pass IWDG stop → jumps to application     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Zone 2: Application (AXI SRAM execution)            │
│                                                     │
│  Application links:                                 │
│    - libsbop-ota.a  (OTA service library)           │
│    - TLS stack (mbedTLS or wolfSSL)                 │
│    - HTTP client                                    │
│    - JSON / CBOR parser                             │
│                                                     │
│  OTA service library API:                           │
│    ota_check_for_update()      → UpdateCheckResult  │
│    ota_download_and_verify()    → Result<(), OTAError>│
│    ota_verify_manifest()        → Result<bool, ...>  │
│    ota_activate_and_reboot()    → Never              │
│    ota_rollback()               → Result<(), ...>     │
│                                                     │
│  OTA library responsibilities:                      │
│    - Authenticate with backend (KD_Auth)            │
│    - Verify backend response signature (BACKEND_AUTH)│
│    - Verify firmware manifest signature using      │
│      embedded public key (in-band, CSM multi-key)  │
│    - Download encrypted image via TLS               │
│    - Verify encrypted blob hash (SHA-256)           │
│    - Write CIPHERTEXT to inactive NOR slot           │
│    - Verify NOR write (read-back + hash re-check)   │
│    - Set slot metadata (VERIFIED status)             │
│    - Trigger system reset to activate               │
│    - NEVER decrypts — plaintext never in NOR         │
└─────────────────────────────────────────────────────┘
```

**Why OTA doesn't belong in the bootloader:**

| Factor | Bootloader OTA | Application Library OTA |
|---|---|---|
| **Code size** | Adds ~20 KB to bootloader (TLS + HTTP + OTA + decrypt) | ~12 KB library (no TLS, no decrypt) — application already has TLS stack |
| **Attack surface** | Network stack in Zone 1 exposes bootloader to remote attacks | Network code in Zone 2, isolated from boot verification |
| **Update reliability** | Bootloader bug = bricked device, no recovery without debugger | Application bug = app crash, bootloader still boots fallback |
| **Resume support** | Bootloader must persist download state across resets | Application can checkpoint and resume after reboot |
| **Rollback decision** | Bootloader must detect boot failure (watchdog timeout) | Application can self-report health before activation |
| **Power failures** | Mid-download power loss = corrupted slot, no recovery context | Application maintains download progress in metadata, resumes |
| **Memory budget** | Bootloader runs in 128 KB flash, 4 KB stack | Application has full 512 KB AXI SRAM + 352 KB SRAM pools |

**libsbop-ota API design:**

```
// OTA service library — linked into application (Zone 2)
// Uses bootloader services via SVC calls for key derivation

typedef enum {
    OTA_NO_UPDATE,
    OTA_UPDATE_AVAILABLE,
    OTA_ALREADY_LATEST,
} ota_check_result_t;

// Check for available update (authenticate + query backend)
ota_error_t ota_check_for_update(
    const char* backend_url,
    uint32_t current_version,
    ota_check_result_t* result
);

// Download encrypted image, verify signatures + hash, write CIPHERTEXT to NOR
// NO DECRYPTION — plaintext never in NOR. Bootloader decrypts at boot.
// Supports resume via HTTP Range if interrupted
ota_error_t ota_download_and_install(
    const char* download_url,
    const uint8_t encrypted_payload_hash[32],  // SHA-256 of IV || AES-CTR(Payload) || InnerSigBlock
    slot_id_t target_slot
);

// Activate new firmware — writes slot metadata, triggers reset
void ota_activate_and_reboot(void) __attribute__((noreturn));

// Rollback to previous firmware version
ota_error_t ota_rollback(void);
```

**Key access from application (Zone 2):**

The application never sees raw key material. K_s (OTA session key) and KD_Auth are derived inside the bootloader's PC-ROP protected SVC handler and loaded directly into CRYP key registers:

```
// Application does NOT decrypt. SVC is needed only for KD_Auth derivation.
// K_s is NOT derived by OTA library — only bootloader performs ECDH at boot time.

// OTA library verifies: backend response signature + manifest signature + encrypted blob hash
// Writes E || IV || ciphertext || InnerSignatureBlock to NOR as-is.
// Plaintext Payload never in NOR — only in AXI SRAM after bootloader decrypt at boot.
```

---

### 3.8 Decision: Factory Programming via DAP-Link + ITCM Flash Loader (Split Pass)

**Decision:** Factory provisioning uses a **DAP-Link SWD probe** and a tiny **ITCM-resident flash loader**. The process is split into **three independent passes** to work within RAM constraints: Pass 1 writes the bootloader (fits in SRAM1), Pass 2 streams the application in 32 KB chunks, Pass 3 provisions security registers and locks OTP.

**Why split passes:**

| Resource | Size | Constraint |
|---|---|---|
| Bootloader | ~45 KB | Fits in any SRAM block (128 KB available) |
| Application | ≤ 512 KB | Exceeds all SRAM blocks except AXI SRAM, which is the execution target |
| AXI SRAM | 512 KB | Cannot hold both bootloader + application simultaneously during programming |
| SRAM1/2/3 total | 352 KB | Not contiguous, max single block 128 KB |
| DTCM | 128 KB | Bootloader stack/data at runtime, free during flash loader |

A single-pass approach (buffer entire application in RAM) is impossible. The flash loader uses a **command-driven chunked streaming** protocol: host writes a 32 KB chunk to SRAM1 via SWD, flash loader writes it to NOR, host writes the next chunk, repeat.

**Flash loader command protocol (ITCM resident, ~800 bytes):**

```
// Command interface — host writes to these addresses via SWD, then resumes MCU
// All addresses in ITCM (0x0000_0x00 range), adjacent to flash loader code

volatile uint32_t CMD_REG    @ 0x0000_0100;  // Command code (host write)
volatile uint32_t CMD_SIZE   @ 0x0000_0104;  // Payload size / chunk index
volatile uint32_t CMD_ADDR   @ 0x0000_0108;  // Target address (flash or NOR offset)
volatile uint32_t CMD_CRC    @ 0x0000_010C;  // CRC-32 of payload buffer
volatile uint32_t CMD_STATUS @ 0x0000_0110;  // Status (loader write, host poll)
volatile uint8_t  CMD_BUF[]  @ 0x0000_0200;  // Payload buffer (host writes data here)

// Command codes
#define CMD_PROGRAM_BOOTLOADER   0x01   // Program internal flash (SRAM1 buffer)
#define CMD_ERASE_NOR_SECTOR     0x02   // Erase one 4 KB NOR sector
#define CMD_WRITE_NOR_CHUNK      0x03   // Write 32 KB chunk to NOR (SRAM1 buffer)
#define CMD_WRITE_SECURITY_REG   0x04   // Write SR1/SR2/SR3 (SR1=0, SR2=1, SR3=2)
#define CMD_LOCK_SECURITY_REG    0x05   // Set OTP lock bits (LB1|LB2|LB3 mask)
#define CMD_VERIFY_CHUNK         0x06   // Read-back + compare NOR chunk
#define CMD_VERIFY_BOOTLOADER    0x07   // Read-back + compare internal flash
#define CMD_DONE                  0xFF   // All operations complete — WFI

// Status codes
#define STATUS_READY              0x00   // Waiting for command
#define STATUS_BUSY               0x01   // Executing command
#define STATUS_OK                 0x2A   // Command succeeded
#define STATUS_ERR_ERASE          0xE1   // NOR erase failed
#define STATUS_ERR_WRITE          0xE2   // NOR write failed
#define STATUS_ERR_VERIFY         0xE3   // Read-back mismatch
#define STATUS_ERR_LOCK           0xE4   // OTP lock failed
```

**Flash loader main loop:**

```
void flash_loader_entry(void) {
    // Init: clock (HSE→400 MHz, PLL2Q→SPI1 80 MHz), SPI1 (80 MHz, polled)
    platform_clock_init_fast();
    spi1_init_master_80mhz();

    while (1) {
        CMD_STATUS = STATUS_READY;

        // Wait for host to write a command
        while (CMD_REG == 0) __WFI();

        CMD_STATUS = STATUS_BUSY;
        uint32_t cmd = CMD_REG;

        switch (cmd) {
        case CMD_PROGRAM_BOOTLOADER:
            // SRAM1 (0x3000_0000) holds bootloader image (~45 KB)
            // CMD_SIZE = bootloader size in bytes
            flash_unlock();
            flash_erase_sectors(0x0800_0000, CMD_SIZE);
            flash_program(0x0800_0000, (uint8_t*)0x3000_0000, CMD_SIZE);
            CMD_STATUS = STATUS_OK;
            break;

        case CMD_ERASE_NOR_SECTOR:
            // CMD_ADDR = NOR sector address (4 KB aligned)
            nor_spi_sector_erase(CMD_ADDR);
            CMD_STATUS = STATUS_OK;
            break;

        case CMD_WRITE_NOR_CHUNK:
            // SRAM1 (0x3000_0000) holds 32 KB chunk
            // CMD_ADDR = NOR target address, CMD_SIZE = chunk size
            nor_spi_page_program(CMD_ADDR, (uint8_t*)0x3000_0000, CMD_SIZE);
            CMD_STATUS = STATUS_OK;
            break;

        case CMD_VERIFY_CHUNK:
            // Read back NOR chunk into SRAM2, compare with SRAM1
            nor_spi_read(CMD_ADDR, (uint8_t*)0x3002_0000, CMD_SIZE);
            if (memcmp((void*)0x3000_0000, (void*)0x3002_0000, CMD_SIZE) == 0)
                CMD_STATUS = STATUS_OK;
            else
                CMD_STATUS = STATUS_ERR_VERIFY;
            break;

        case CMD_VERIFY_BOOTLOADER:
            // Compare internal flash against SRAM1 buffer
            if (memcmp((void*)0x0800_0000, (void*)0x3000_0000, CMD_SIZE) == 0)
                CMD_STATUS = STATUS_OK;
            else
                CMD_STATUS = STATUS_ERR_VERIFY;
            break;

        case CMD_WRITE_SECURITY_REG:
            // CMD_ADDR = register index (0=SR1, 1=SR2, 2=SR3)
            // CMD_BUF[] = data to write (up to 256 bytes, cross-vendor SR size limit)
            nor_spi_write_security_register(CMD_ADDR, CMD_BUF, CMD_SIZE);
            CMD_STATUS = STATUS_OK;
            break;

        case CMD_LOCK_SECURITY_REG:
            // CMD_ADDR = lock bit mask (bit 0=LB1, bit 1=LB2, bit 2=LB3)
            nor_spi_lock_security_registers(CMD_ADDR);
            CMD_STATUS = STATUS_OK;
            break;

        case CMD_DONE:
            CMD_STATUS = STATUS_OK;
            while (1) __WFI();  // Done — wait for host to reset MCU
        }

        CMD_REG = 0;  // Clear command register
    }
}
```

**Manufacturing provisioning sequence (split passes):**

```
Pass 1 — Program Bootloader (~5 seconds)
──────────────────────────────────────────
Step  Host (Python + pyOCD)                    DUT
────  ──────────────────────────────────────── ───────────────────────
  1   pyOCD: halt MCU, connect SWD             Halted in reset
  2   pyOCD: upload flash_loader.bin to ITCM    ITCM loaded (~800 B)
  3   pyOCD: upload bootloader.bin to SRAM1     SRAM1 holds ~45 KB
  4   pyOCD: write CMD=0x01, SIZE=45KB, resume  Flash loader runs
  5                                             Erase + program 0x0800_0000
  6                                             CMD_STATUS = 0x2A (OK)
  7   pyOCD: upload verify cmd, SIZE=45KB       Read-back verify
  8                                             CMD_STATUS = 0x2A (OK)
  9   pyOCD: halt MCU                           Halted

Pass 2 — Program Application (~30 seconds for 512 KB)
──────────────────────────────────────────────────────
 10   pyOCD: resume flash loader                Waiting for commands
       // Erase all Slot A sectors (256 × 4 KB)
 11   for each sector 0x000000..0x100000:
          pyOCD: CMD=ERASE_NOR_SECTOR, ADDR=s
          resume → wait STATUS_OK → halt
       // Stream application in 32 KB chunks
 12   for each 32 KB chunk of application.bin:
          pyOCD: upload chunk to SRAM1
          pyOCD: CMD=WRITE_NOR_CHUNK, ADDR=offset
          resume → wait STATUS_OK → halt
          pyOCD: CMD=VERIFY_CHUNK, ADDR=offset
          resume → wait STATUS_OK → halt
          offset += 32 KB

Pass 3 — Provision Security Registers + Lock OTP
─────────────────────────────────────────────────
       // Host-side HSM computes commitments
 13   pyOCD: read MCU UID (0x1FF1_E800 via SWD)
       pyOCD → flash loader → CMD to read NOR UID (4BH cmd)
       pyOCD: read NOR UID from SRAM1
       pyOCD → HSM callback:
           KD = HKDF-SHA-256(KR, MCU_UID || NOR_UID)
           KD_commitment = HMAC-SHA-256(KD, "SBOP-DEVICE-COMMIT")
           binding = HMAC-SHA-256(NOR_UID, MCU_UID)
 14   pyOCD: write SR1 data to CMD_BUF
           (version=1, slot=A, flags=0)
       pyOCD: CMD=WRITE_SECURITY_REG, ADDR=0
       resume → wait STATUS_OK → halt
 15   pyOCD: write KD_commitment to CMD_BUF
       pyOCD: CMD=WRITE_SECURITY_REG, ADDR=1
       resume → wait STATUS_OK → halt
 16   pyOCD: write NOR-MCU binding to CMD_BUF
       pyOCD: CMD=WRITE_SECURITY_REG, ADDR=2
       resume → wait STATUS_OK → halt
 17   pyOCD: CMD=LOCK_SECURITY_REG, ADDR=(LB1|LB2|LB3)
       resume → wait STATUS_OK → halt
 18   pyOCD: CMD=DONE → resume                   WFI

Post-Programming — Option Bytes + Final Verify
───────────────────────────────────────────────
 19   pyOCD: halt MCU
 20   pyOCD: program Option Bytes:
           ROP = Level 1, PC-ROP sectors, BFB2 = 0
 21   pyOCD: reset MCU                           Full boot:
                                                    Bootloader verifies Slot A
                                                    Copies app to AXI SRAM
                                                    Executes application
 22   pyOCD: disconnect SWD                      DUT ships
```

**Key design points:**

| Point | Detail |
|---|---|
| **Bootloader buffer** | SRAM1 (128 KB) — bootloader is ~45 KB, fits easily |
| **Application streaming** | 32 KB chunks into SRAM1 (128 KB). SRAM2 (128 KB) used as read-back buffer for verify. 512 KB / 32 KB = 16 iterations. |
| **KD commitment** | Computed on host-side HSM, not on-device. KR never leaves HSM. Flash loader only writes pre-computed commitment and binding. |
| **NOR UID** | Read via flash loader SPI command (4BH), returned to host via SRAM1 buffer. |
| **OTP lock** | Performed only after all three SRs are written and verified. Lock is irreversible — verified by read-back before locking. |
| **Flash loader** | ~800 bytes in ITCM. No internal flash used. Disappears after reset. Zero footprint in deployed bootloader. |

**Why this approach:**

| Factor | DAP-Link + ITCM (Split) | Single-Pass Buffer | Y-Modem Factory |
|---|---|---|---|
| **Bootloader size** | 0 bytes added | 0 bytes added | +4 KB |
| **RAM required** | 128 KB (SRAM1) + 128 KB (SRAM2 for verify) | 512 KB+ contiguous | 1 KB |
| **Application streaming** | 32 KB chunks, 16 iterations | One-shot buffer (impossible on this platform) | 1K frames, 512 iterations |
| **Speed** | ~35 seconds for 512 KB | N/A | Minutes |
| **Automation** | Full Python scripting | N/A | Manual terminal |
| **Multi-DUT** | 4-8 per station | N/A | 1 per station |
| **Root key security** | HSM on host, commitment sent to DUT | HSM on host | HSM on host |
| **Bootloader stays minimal** | Yes — flash loader is external | Yes | No — Y-Modem in bootloader |

---

### 3.9 Decision: Y-Modem Recovery via USART1

**Decision:** The bootloader includes a minimal Y-Modem receiver for rescue/recovery. It is only entered when both slots are INVALID (boot fails) or when a recovery command is received during the FAILSAFE loop. It is **not** used for factory provisioning.

**Rationale:**

Y-Modem in the bootloader is a safety net, not a manufacturing path. If both slots are corrupted (e.g., power loss during OTA), the bootloader enters FAILSAFE and waits for Y-Modem recovery on USART1. A technician connects a terminal, sends a signed firmware image, and the bootloader verifies and writes it to a slot.

**Y-Modem Rx implementation (bootloader, ~3 KB):**

```
// Minimal Y-Modem receiver — 1K CRC variant, no streaming
// USART1: 115200 bps (recovery), 96 MHz PCLK
// Buffer: single 1K frame + header parsing in DTCM

function ymodem_receive_firmware() -> Result<(), BootError>:
    // Wait for Y-Modem start (0x43 'C' poll, 60s timeout)
    if !ymodem_wait_start(60_000_ms):
        return Err(ERR-RECOV-TIMEOUT)

    // Receive file header (filename, size)
    let header = ymodem_receive_frame(0)?
    let image_size = parse_ymodem_header(header)

    // Validate image size against slot capacity
    if image_size == 0 or image_size > SLOT_MAX_SIZE:
        return Err(ERR-BOOT-PARSE-004)

    // Select target slot (first EMPTY or INVALID)
    let target_slot = select_recovery_slot()?

    // Receive and write firmware
    // IWDG is active (Pass 1, 800 ms timeout). Y-Modem at 115200 bps:
    // each 1K frame takes ~89 ms on the wire, plus NOR write time.
    // Refresh IWDG every frame to prevent watchdog reset mid-recovery.
    let frame_num = 1
    let bytes_written = 0
    loop:
        IWDG_REFRESH()
        let frame = ymodem_receive_frame(frame_num)?
        let data = frame.payload[0..min(frame.payload_len, image_size - bytes_written)]

        // NOR sector erase before write (if crossing sector boundary)
        if bytes_written % NOR_SECTOR_SIZE == 0:
            IWDG_REFRESH()  // Sector erase ~45 ms — refresh before
            nor_spi_erase_sector(target_slot.base + bytes_written)

        IWDG_REFRESH()  // Refresh before page program
        storage_write(target_slot.base + bytes_written, data)
        bytes_written += data.len
        if bytes_written >= image_size:
            break
        frame_num += 1

    // Verify uploaded image
    let image_data = storage_read_slot(target_slot)
    let header = parse_header(image_data)?
    let payload = image_data[header.header_size .. header.header_size + header.image_size]

    let hash = sha256(payload)
    assert(constant_time_compare(hash, header.hash, 32), ERR-BOOT-CRYPTO-002)

    // Verify signature using multi-key CSM verification (reuses boot_verify_and_execute_slot logic)
    // Extracts key_index + public_key from sig_block, verifies fingerprint against OTP,
    // checks revocation bitmap, then verifies Ed25519(sig_block.public_key)
    let sig_block = parse_signature_block(image_data, header)
    assert(key_fingerprint_verify(sig_block.key_index, sig_block.public_key), ERR-BOOT-CRYPTO-004)
    assert(!key_is_revoked(sig_block.key_index), ERR-BOOT-CRYPTO-004)
    let sig_valid = ed25519_verify(
        concat(image_data[0..header.header_size], payload),
        sig_block.signature,
        sig_block.public_key
    )
    assert(sig_valid, ERR-BOOT-CRYPTO-001)

    // Mark slot VERIFIED and trigger boot
    storage_set_slot_status(target_slot, VERIFIED)
    storage_set_boot_target(target_slot)
    system_reset()
```

**Y-Modem entry conditions:**

```
Bootloader enters Y-Modem recovery when:
  ├── Both slots INVALID or CORRUPT → automatic entry
  ├── FAILSAFE loop receives RECOVERY_OTA_INITIATE via USART1
  └── Factory fresh device (both slots EMPTY → rapid Y-Modem start,
      but factory should use DAP-Link instead)

Y-Modem parameters:
  ├── USART1: PA9/PA10, Low speed, AF7
  ├── Baud: 115200 (recovery), 921600 (optional fast recovery)
  ├── Protocol: Y-Modem 1K (CRC-16), no streaming
  ├── Timeout: 60s for start, 10s per frame
  └── Max retries: 5 per frame
```

**Bootloader size impact:**

| Addition | Size |
|---|---|
| Y-Modem state machine + frame parser | 1.5 KB |
| USART1 RX/TX driver (polled) | 0.5 KB |
| Recovery slot selection + write loop | 0.5 KB |
| Frame CRC-16 verify | 0.25 KB |
| **Total Y-Modem** | **~3 KB** |

---

### 3.10 Decision: OTA Package Format and Dual-Verification Chain

**Decision:** Every firmware image carries an Ed25519 digital signature. The OTA library verifies the signature and integrity of the **encrypted** package — it never decrypts and never writes plaintext to NOR flash. The bootloader decrypts on-the-fly to volatile AXI SRAM at boot time. This prevents the power-off attack where an attacker cuts power after OTA to extract plaintext firmware from NOR flash.

**Security rationale:** If OTA decrypts before writing, plaintext firmware resides in NOR flash from OTA completion until the next boot — a window of minutes to hours. An attacker with physical access can cut power during this window and extract the plaintext image via chip-off or bus sniffing. By keeping firmware encrypted in NOR at all times and decrypting only to volatile AXI SRAM, plaintext exists only for the ~300 ms boot window and vanishes on power loss.

#### 3.10.1 Package Format

**Step 1 — Backend builds plaintext firmware image (once per firmware version):**

```
ImageHeader (80 B)
├── magic              : u32  = 0x53424F50 ("SBOP")
├── header_version      : u16  = 1
├── header_size         : u16  = 80
├── image_size          : u32  = payload length (plaintext)
├── firmware_version    : u32  = semantic version (BCD)
├── board_id            : u16  = target board SKU
├── flags               : u32  = FLAG_ENCRYPTED (always set for OTA)
├── hash                : u8[32] = SHA-256(Payload)  ← plaintext hash (bootloader verifies after decrypt)
└── reserved            : u8[10] = 0

Payload (≤ 512 KB)
└── Application binary (plaintext)

InnerSignatureBlock (116 B)
├── algorithm     : u8   = 0x02 (Ed25519)
├── key_index     : u8   = 0x00-0x03 (key slot) or 0xFF (legacy single-key)
├── sig_length    : u16  = 64
├── reserved      : u8[4] = 0
├── public_key    : u8[32] = Ed25519 KI_public[key_index] (verified against OTP fingerprint)
├── signature     : u8[72] = Ed25519(KI_private[key_index], SHA-256(ImageHeader || Payload))
└── crc32         : u32

firmware_image = ImageHeader || Payload || InnerSignatureBlock
```

**Step 2 — Backend X25519 hybrid encrypt (once for all devices):**

```
key_index = active OTA key pair slot (0..3, backend HSM selects)

e           = CSPRNG(32 bytes)                                   ← Ephemeral X25519 scalar
E           = e * G                                              ← Ephemeral public key (32 B, sent to device)
S           = X25519(e, OTA_public[key_index])                    ← ECDH shared secret
K_s         = HKDF-SHA-256(S, "SBOP-OTA-ENC", 32)                ← AES-256 session key
iv          = CSPRNG(16 bytes)                                   ← Random IV
encrypted_payload = AES-256-CTR(K_s, iv, Payload)

// InnerSignatureBlock is OUTSIDE the encryption envelope.
// ImageHeader is OUTSIDE the encryption envelope (OTA needs metadata).
// Ephemeral public key E and IV are plaintext, OUTSIDE the envelope.
// Only Payload is encrypted.
// enc_key_index is written to ImageHeader.reserved[0].

header.reserved[0] = key_index
OTA wire package = ImageHeader || E || iv || encrypted_payload || InnerSignatureBlock

// Same package for ALL devices. Backend encrypts ONCE per firmware release.
// Per-device at-rest uniqueness comes from KD re-encrypt on first boot (Stage 2).
// Package size: 64 + 32 + 16 + N + 116 = N + 228 bytes
```

**Why SignatureBlock is outside encryption:**

The OTA library must verify the Ed25519 signature without decrypting. Since the signature covers `ImageHeader || Payload`, and `ImageHeader` is plaintext, the OTA library needs `Payload` plaintext to verify... which it doesn't have.

Solution: the backend computes a **signed manifest** that includes the encrypted blob's hash. The manifest is signed once with KI_private. The OTA library verifies the manifest signature + encrypted blob hash — no decryption needed.

**Step 3 — Backend creates signed update manifest (once per firmware version):**

```
// encrypted_blob = E || iv || encrypted_payload || InnerSignatureBlock
// Hash of the complete encrypted package after ImageHeader.
// Includes E (ephemeral public key) so Gate 2 can verify it hasn't been tampered.

UpdateManifest (signed once per firmware version):
├── firmware_version      : u32
├── board_id              : u16
├── image_size            : u32
├── payload_hash          : u8[32]  = SHA-256(Payload_plaintext)
│                                    ← bootloader verifies this after decrypt
├── header_hash           : u8[32]  = SHA-256(ImageHeader)
├── encrypted_blob_hash   : u8[32]  = SHA-256(E || iv || encrypted_payload || InnerSigBlock)
│                                    ← Gate 1 verifies this (no decryption needed)
├── ota_key_index         : u8      = active OTA key pair (0..3)
├── min_bootloader_ver    : u32
├── min_battery_pct       : u8
└── reserved              : u8[22]

ManifestSignatureBlock (116 B)
├── algorithm     : u8   = 0x02 (Ed25519)
├── key_index     : u8   = 0x00-0x03 or 0xFF
├── sig_length    : u16  = 64
├── reserved      : u8[4] = 0
├── public_key    : u8[32] = Ed25519 KI_public[key_index]
├── signature     : u8[72] = Ed25519(KI_private[key_index], SHA-256(UpdateManifest))
└── crc32         : u32
```

The manifest + its signature are the same for all devices. KI_private signs once.

**Step 4 — Per-request OTA response (signed by backend at request time):**

```
BackendUpdateResponse (per-device, per-request):
├── UpdateManifest (from Step 3)
├── ManifestSignatureBlock (from Step 3)
├── download_url           : UrlString
├── encrypted_blob_size    : u32  (= 32 + 16 + encrypted_payload_size + 116)
└── expires_at             : u32  (token expiration)

BackendResponseSignature (64 B)
└── Ed25519(BACKEND_AUTH_private, SHA-256(BackendUpdateResponse))
```

The backend signs the response with BACKEND_AUTH per-request. The encrypted_blob_hash in the manifest is the same for all devices (same E, IV, ciphertext). BACKEND_AUTH signs metadata, not firmware.

#### 3.10.2 Verification Gate 1: OTA Library (Zone 2, pre-write)

**OTA library does NOT decrypt. It verifies two signatures, both over non-secret data.**

```
// libsbop-ota: ota_download_and_install()
// Runs in application context. NO decryption. NO plaintext in NOR.

function ota_download_and_install(update_info, target_slot):

    // ── Step 1: Verify backend response signature ──
    // This proves the response (including encrypted_payload_hash) is authentic
    backend_sig_valid = ed25519_verify(
        data = serialize(update_info.BackendUpdateResponse),
        sig  = update_info.BackendResponseSignature,
        key  = BACKEND_AUTH_PUBLIC
    )
    assert(backend_sig_valid, ERR-OTA-AUTH-001)

    // ── Step 2: Verify firmware manifest signature (KI_private[key_index], signed once) ──
    // This proves the manifest (version, board_id, payload_hash) is authentic
    // The manifest signature block carries the public key + key_index in-band
    let sig_block = parse_manifest_signature_block(update_info.ManifestSignatureBlock)
    manifest_sig_valid = ed25519_verify(
        data = serialize(update_info.UpdateManifest),
        sig  = sig_block.signature,
        key  = sig_block.public_key   // Embedded in ManifestSignatureBlock (NXP CSM-style)
    )
    assert(manifest_sig_valid, ERR-OTA-AUTH-001)
    // Note: OTA library (Zone 2) uses the embedded public key directly.
    // Key fingerprint verification + revocation check are enforced by Gate 2 (bootloader, Zone 1)
    // which has access to SR1 OTP.

    // ── Step 3: Board ID check ──
    board_id = board_info_read_board_id()
    assert(update_info.UpdateManifest.board_id == board_id, ERR-BOOT-PARSE-008)

    // ── Step 4: Version check ──
    assert(update_info.UpdateManifest.firmware_version > current_version, ERR-OTA-AUTH-003)

    // ── Step 4b: Size check (prevents NOR slot overflow) ──
    // ImageHeader(80) + E(32) + IV(16) + encrypted_payload + InnerSignatureBlock(116) ≤ SLOT_SIZE
    max_image_size = SLOT_SIZE - 80 - 32 - 16 - 116
    assert(update_info.UpdateManifest.image_size <= max_image_size, ERR-OTA-DOWNLOAD-003)

    // ── Step 5: Download ENCRYPTED package ──
    // Package: ImageHeader || E || IV || encrypted_payload || InnerSignatureBlock
    // OTA reads ImageHeader (plaintext), passes the rest through as opaque bytes
    download_handle = http_download_begin(update_info.download_url)

    // Read ImageHeader (80 B, plaintext)
    header_bytes = http_read_exact(download_handle, 80)
    header = parse_header(header_bytes)
    assert(header.magic == 0x53424F50, ERR-BOOT-PARSE-001)
    assert(header.firmware_version == update_info.UpdateManifest.firmware_version,
           ERR-OTA-DOWNLOAD-002)

    // Verify ImageHeader hash matches manifest
    computed_header_hash = sha256(header_bytes)
    assert(constant_time_compare(computed_header_hash,
           update_info.UpdateManifest.header_hash, 32), ERR-OTA-DOWNLOAD-002)

    // Download rest: E (32 B) || IV (16 B) || encrypted_payload (N B) || InnerSignatureBlock (116 B)
    // OTA treats this as an opaque blob — NO DECRYPTION, NO ECDH
    encrypted_rest = http_read_remaining(download_handle)
    expected_enc_size = 32 + 16 + update_info.encrypted_blob_size + 116
    assert(len(encrypted_rest) == expected_enc_size, ERR-OTA-DOWNLOAD-002)

    // ── Step 6: Verify encrypted blob hash against signed manifest ──
    // encrypted_blob = E || IV || encrypted_payload || InnerSignatureBlock
    // This proves the entire encrypted package (including E) is exactly what the backend intended
    computed_enc_hash = sha256(encrypted_rest)  // E || IV || ciphertext || InnerSignatureBlock
    assert(constant_time_compare(computed_enc_hash,
           update_info.UpdateManifest.encrypted_blob_hash, 32), ERR-OTA-DOWNLOAD-002)

    // ── Step 7: Write ENCRYPTED package to NOR ──
    // Plaintext NEVER touches NOR flash
    storage_write(target_slot.base + 0,   header_bytes)       // 80 B, plaintext header
    storage_write(target_slot.base + 80,  encrypted_rest)     // E + IV + ciphertext + InnerSigBlock
    // Total written to NOR: ~80 + 32 + 16 + 512 KB + 116 B ≈ 512.2 KB (mostly ciphertext)

    // ── Step 8: Verify NOR write integrity ──
    nor_data = storage_read_slot(target_slot)
    nor_hash = sha256(nor_data[80..])  // Skip header, hash the encrypted portion
    assert(constant_time_compare(nor_hash, computed_enc_hash, 32), ERR-OTA-INSTALL-002)

    // ── Step 9: Mark slot VERIFIED ──
    storage_set_slot_status(target_slot, VERIFIED)
    storage_set_slot_version(target_slot, header.firmware_version)

    return OK
```

**What Gate 1 proves (all without decryption):**

| Check | What it proves | Key used |
|--------|---------------|----------|
| BackendResponseSignature | The response is from the real backend | BACKEND_AUTH_PUBLIC |
| ManifestSignatureBlock | The firmware manifest is from the real signer | KI_public[key_index] (embedded in sig block) |
| board_id match | Firmware is for this board SKU | (plaintext compare) |
| version > current | Not a downgrade | (plaintext compare) |
| header_hash match | The downloaded ImageHeader matches manifest | SHA-256 |
| encrypted_blob_hash match | E || IV || ciphertext || InnerSigBlock hasn't been tampered | SHA-256 |

#### 3.10.3 Verification Gate 2: Bootloader (Zone 1, pre-execute)

**Bootloader performs X25519 ECDH with OTA_private[enc_key_index], derives K_s, decrypts to AXI SRAM, verifies, re-encrypts with KD (per-device), writes back to NOR. On subsequent boots, decrypts directly with KD.**

After re-encryption with KD (the permanent device key), the image is always at-rest encrypted with per-device KD. Every subsequent boot decrypts with KD directly — no ECDH needed. OTA_private[i] are fixed X25519 scalars in bootloader .rodata, protected by PC-ROP. Extracting them enables decryption only (cannot encrypt fake firmware without OTA_public[i] and KI_private).

```
// Bootloader: sbop_boot() — Phase 5 through Phase 11

function sbop_boot():
    // ... Phases 1-4: INIT, SELECT_SLOT, LOAD_IMAGE, PARSE_HEADER ...

    // ImageHeader is plaintext in NOR — parse it directly
    header = parse_header(image_data[0..80])

    // Verify HMAC over header fields (SPI bus integrity)
    let hmac_input = image_data[0..56]  // All fields before header_hmac
    let expected_hmac = hmac_sha256(kd, hmac_input)[0..16]  // Truncated to 128 bits
    assert(constant_time_compare(image_data[56..72], expected_hmac, 16), ERR-BOOT-PARSE-010)
    // FIH: redundant HMAC verify
    assert(constant_time_compare(image_data[56..72], expected_hmac, 16), ERR-BOOT-STATE-003)

    // encrypted_rest = E (32 B) || IV (16 B) || AES-CTR(Payload) || InnerSignatureBlock (116 B)
    encrypted_rest = image_data[80..]
    E  = encrypted_rest[0..32]       // Ephemeral public key (32 B)
    iv = encrypted_rest[32..48]       // IV (16 B)
    ciphertext_with_sig = encrypted_rest[48..]  // AES-CTR(Payload) || InnerSigBlock

    // ── Phase 5: ECDH + Decrypt + Verify Signature ──
    state = VERIFY_SIGNATURE

    // Determine which key to use for decryption
    if header.flags & FLAG_OTA_PENDING:
        // ── First boot after OTA: X25519 ECDH + AES decrypt ──
        enc_key_index = header.reserved[0]                     // Read from ImageHeader

        // Verify OTA key fingerprint (against SR2 OTP)
        OTA_private = OTA_PRIVATE_TABLE[enc_key_index]         // From bootloader .rodata
        OTA_public  = X25519(OTA_private, G)                   // Derive public key
        let expected_fp = nor_spi_read_sr1(OTA_KEY_FINGERPRINT_BASE + enc_key_index * 32, 32)
        let actual_fp   = sha256(OTA_public)
        assert(constant_time_compare(actual_fp, expected_fp, 32), ERR-BOOT-CRYPTO-004)

        // Check OTA key revocation
        let ota_revocation = nor_spi_read_sr1_u16(OTA_KEY_REVOCATION_OFFSET)
        assert((ota_revocation >> enc_key_index) & 1 == 1, ERR-BOOT-CRYPTO-004)

        // X25519 ECDH: S = private * E = private * e * G
        S = X25519(OTA_private, E)                             // ECDH shared secret (32 B)
        K_s = HKDF_SHA256(S, "SBOP-OTA-ENC", 32)               // AES-256 session key
        secure_zeroize(S, 32)                                   // Shared secret no longer needed

        // AES-256-CTR decrypt → AXI SRAM (volatile only)
        plaintext_with_sig = aes_ctr_decrypt(K_s, iv, ciphertext_with_sig)
        secure_zeroize(K_s, 32)                                 // Session key no longer needed
    else:
        // ── Normal boot: KD decrypt ──
        plaintext_with_sig = aes_ctr_decrypt(kd, iv, ciphertext_with_sig)

    // plaintext_with_sig = Payload || InnerSignatureBlock
    // Written to AXI SRAM — volatile, lost on power-off

    // Parse decrypted data
    payload_len = header.image_size
    payload = plaintext_with_sig[0..payload_len]
    inner_sig_block = parse_signature_block(plaintext_with_sig, payload_len)

    // ── Signing key fingerprint verification (NXP CSM-style multi-key) ──
    let sig_key_index = inner_sig_block.key_index
    let public_key = inner_sig_block.public_key

    // Verify signing key is not revoked
    let sig_revocation = nor_spi_read_sr1_u16(SIG_KEY_REVOCATION_OFFSET)
    assert((sig_revocation >> sig_key_index) & 1 == 1, ERR-BOOT-CRYPTO-004)

    // Verify signing public key fingerprint matches OTP
    let sig_expected_fp = nor_spi_read_sr1(SIG_KEY_FINGERPRINT_BASE + sig_key_index * 32, 32)
    let sig_actual_fp   = sha256(public_key)
    assert(constant_time_compare(sig_actual_fp, sig_expected_fp, 32), ERR-BOOT-CRYPTO-004)

    // Verify Ed25519 signature (Gate 2) using embedded public key
    sig_input = concat(image_data[0..80], payload)  // ImageHeader || Payload
    sig_valid = ed25519_verify(sig_input, inner_sig_block.signature, public_key)
    assert(sig_valid, ERR-BOOT-CRYPTO-001)

    // FIH: redundant verify
    sig_valid2 = ed25519_verify(sig_input, inner_sig_block.signature, public_key)
    assert(sig_valid == sig_valid2, ERR-BOOT-STATE-002)

    // ── Phase 6: Verify Integrity ──
    state = VERIFY_INTEGRITY
    computed_hash = sha256_hw(payload)              // HW HASH (primary)
    computed_hash_sw = sha256_sw(payload)           // Software SHA-256 cross-check
    // Both computation paths MUST agree — detects HW HASH glitch or FIH attack
    assert(constant_time_compare(computed_hash, computed_hash_sw, 32), ERR-BOOT-CRYPTO-005)
    assert(constant_time_compare(computed_hash, header.hash, 32), ERR-BOOT-CRYPTO-002)
    // FIH redundant compare (fault injection defense)
    assert(constant_time_compare(computed_hash, header.hash, 32), ERR-BOOT-CRYPTO-002)

    // ── Phase 6b: Re-encrypt with KD (if first boot after OTA) ──
    if header.flags & FLAG_OTA_PENDING:
        // Image currently encrypted with K_s (ECDH-derived, shared across all devices).
        // Re-encrypt with KD (per-device key) for subsequent boots.
        //
        // WRITE-BEFORE-ERASE PATTERN:
        // Never erase the active slot until the re-encrypted data is verified on disk.
        // The slot is 1 MB but the image is ≤512 KB — use the upper half (≥512 KB) as a
        // staging area. Staging area is pre-erased (0xFF) since it's never been written.

        iv_dev = crypto_random_bytes(16)
        payload_with_sig = concat(payload, inner_sig_block)  // Payload + InnerSigBlock
        encrypted_dev = aes_ctr_encrypt(kd, iv_dev, payload_with_sig)

        // Build new header: copy original, update flags, update hash for KD-encrypted payload
        new_header = header
        new_header.flags &= ~FLAG_OTA_PENDING
        new_header.hash = sha256(encrypted_dev)  // Hash of KD-encrypted payload
        new_header.header_hmac = hmac_sha256(kd, serialize(new_header)[0..56])[0..16]

        // Staging offset: half of slot (512 KB = 0x8_0000)
        staging_base = boot_slot.base + SLOT_STAGING_OFFSET  // 0x8_0000

        // Step 1: Write KD-encrypted data to staging area (pre-erased NOR — no erase needed)
        nor_write(staging_base + 0,   serialize(new_header))   // 80 B, updated header
        nor_write(staging_base + 80,  iv_dev)                   // 16 B, new IV
        nor_write(staging_base + 96,  encrypted_dev)            // KD-encrypted payload + sig

        // Step 2: Verify staging write integrity before erasing original
        staging_data = nor_read(staging_base, 80 + 16 + len(encrypted_dev))
        assert(sha256(staging_data[96..]) == new_header.hash, ERR-STOR-WRITE-001)

        // Step 3: Mark metadata — staging is ready
        metadata_set_re_encrypt_state(RE_STATE_STAGED)

        // Step 4: NOW it's safe to erase the original area (0..staging_offset)
        nor_erase_sector_range(boot_slot.base, staging_base - 1)

        // Step 5: Copy from staging to slot base (or just leave at staging and update pointer)
        // For simplicity, copy staging → slot base so bootloader always reads from slot base
        nor_write(boot_slot.base + 0,   staging_data)  // Entire new image

        // Step 6: Verify final write
        final_data = nor_read(boot_slot.base, len(staging_data))
        assert(constant_time_compare(final_data, staging_data, len(staging_data)),
               ERR-STOR-WRITE-001)

        // Step 7: Commit — clear re-encrypt state
        metadata_set_re_encrypt_state(RE_STATE_IDLE)

        // Staging area will be erased on next OTA write to this slot (or lazily)

    // ... Phases 7-10: CHECK_VERSION, COMMIT_VERSION, MARK_ACTIVE, LOCK_BOOT ...

    // ── Phase 11: Execute (payload already in AXI SRAM) ──
    state = EXECUTE
    // Payload is already in AXI SRAM from Phase 5 decrypt
    // (or will be if we deferred copy — in practice, stream directly to AXI SRAM)

    // Zeroize DTCM workspace
    secure_zeroize(plaintext_data, len(plaintext_data))

    // Two-pass boot: IWDG cannot be software-disabled. Use RTC magic + system reset
    // so Pass 2 jumps to the application with IWDG stopped (see §3.13).
    RTC->BKP0R = IWDG_RESET_MAGIC   // 0x49574447
    NVIC_SystemReset()              // System reset → IWDG stops → Pass 2 → jump to app
```

**What Gate 2 proves:**

| Check | What it proves | Key used |
|--------|---------------|----------|
| OTA key fingerprint verify | SHA-256(OTA_private[i] * G) matches SR2 OTP fingerprint | SHA-256 + X25519 + OTP compare |
| OTA key revocation check | The OTA encryption key slot has not been revoked | SR1 OTA revocation bitmap |
| Signing key fingerprint verify | The embedded signing public key matches the OTP fingerprint for sig_key_index | SHA-256 + OTP compare |
| Signing key revocation check | The signing key slot has not been revoked in SR1 | SR1 signing revocation bitmap |
| Ed25519 verify | Payload was signed by KI_private[sig_key_index] (the real firmware signer) | KI_public[sig_key_index] (embedded in SigBlock) |
| FIH redundant verify | No glitch on the first verification | KI_public[sig_key_index] (repeated) |
| SHA-256 HW (payload) == header.hash | Decrypted payload matches the hash in the signed header | (SHA-256 HW) |
| SHA-256 SW cross-check | HW HASH result matches software SHA-256 (detects HW HASH glitch / FIH) | (SHA-256 SW) |
| Re-encrypt with KD | Image now permanently stored with per-device KD | KD |

**Why re-encrypt with KD:**

```
Before re-encrypt (first boot after OTA):
  NOR: ImageHeader(80B) || E(32B) || iv(16B) || AES-CTR(K_s, iv, Payload) || InnerSigBlock(116B)
  K_s = HKDF-SHA-256(X25519(OTA_private[i], E), "SBOP-OTA-ENC")
  → OTA_private[i] must be available in Zone 1 (PC-ROP protected)
  → E is plaintext in NOR (ephemeral public key, public info)
  → Encrypted portion: ~512 KB; free space (upper 512 KB): erased (0xFF)

Write-before-erase (Phase 6b, first boot only):
  Step 1: Write KD-encrypted data to staging area (slot_base + 0x8_0000):
          staging: ImageHeader(80B) || IV_dev(16B) || AES-CTR(KD, IV_dev, Payload) || InnerSigBlock(116B)
  Step 2: Verify staging write integrity
  Step 3: metadata.re_encrypt_state = STAGED
  Step 4: Erase original area (0x0_0000..0x7_FFFF)
  Step 5: Copy staging → slot base
  Step 6: Verify final write
  Step 7: metadata.re_encrypt_state = IDLE
  → If power fails at any step, on next boot the bootloader checks re_encrypt_state
    and recovers: STAGED → re-copy from staging; IDLE → normal boot

After re-encrypt (all subsequent boots):
  NOR: ImageHeader(80B) || IV_dev(16B) || AES-CTR(KD, IV_dev, Payload) || InnerSigBlock(116B)
  → Decrypt with KD directly — per-device unique at-rest encryption
  → KD is always available (permanent device key in secure element)
  → E and iv are dropped; replaced with single IV_dev
  → Staging area available for next OTA cycle

Benefits:
  - Backend encrypts once per firmware release (X25519 ECDH, not per-device)
  - Per-device at-rest encryption via KD re-encrypt on first boot
  - Simplified backend: no per-device encryption, no key lookup
  - Simplified boot: always use KD for decrypt after re-encrypt (no conditional ECDH)
  - Consistent at-rest format: every slot always encrypted with KD after re-encrypt
  - Power-loss safe: write-before-erase with staging area — no brick window
```

#### 3.10.4 Full Verification Flow

```
Backend (once per firmware version):
  │
  │  KI_private[key] →  Ed25519_sign(UpdateManifest)         ← Manifest sig (once, key_index in sig block)
  │  KI_private[key] →  Ed25519_sign(ImageHeader || Payload) ← Inner sig (once, key_index in sig block)
  │                       │                                    Both carry public key + key_index in-band
  │  e = CSPRNG(32)       │
  │  E = e * G            │
  │  S = X25519(e, OTA_public[idx])                          ← ECDH shared secret
  │  K_s = HKDF(S)    →  AES-256-CTR(K_s, IV, Payload)       ← Same ciphertext for ALL devices
  │                       │
  │  BACKEND_AUTH     → Ed25519_sign(BackendUpdateResponse)  ← Response sig (per-request)
  │                       │
  └───────────────────────┼── TLS 1.3 ──→  Device
                          │
Device (Zone 2: OTA Library)                    Device (Zone 1: Bootloader)
  │                                               │
  │  ╔══ GATE 1 (no decryption) ══╗              │
  │  ║ 1. Verify BackendResponse  ║              │
  │  ║    signature (BACKEND_AUTH)║              │
  │  ║ 2. Verify Manifest         ║              │
  │  ║    signature using embedded║              │  ← KI_public[key] from ManifestSigBlock
  │  ║    public key (in-band)    ║              │
  │  ║ 3. Board ID + version check║              │
  │  ║ 4. SHA-256(E || IV ||      ║              │
  │  ║    ciphertext || InnerSig) ║              │
  │  ║    == manifest.enc_hash    ║              │
  │  ║ 5. Write CIPHERTEXT to NOR ║              │
  │  ║    (K_s-encrypted)         ║              │
  │  ╚════════════════════════════╝              │
  │     ↓                                         │
  │  Slot status = VERIFIED                       │
  │  System reset                                  │
  │                                               │
  └────────────────── reset ──────────────────────┘
                                                  │
                    ┌─────────────────────────────┘
                    │
                    ▼
              ╔══ GATE 2 (bootloader, first boot after OTA) ══╗
              ║ 1. Read enc_key_index from ImageHeader        ║
              ║ 2. pub = OTA_private[idx] * G                 ║  ← Derive public key
              ║ 3. SHA-256(pub) == SR2.ota_fp[idx] (OTP)     ║  ← OTA key auth
              ║ 4. Check SR2.ota_revocation[idx] == 1         ║  ← Key not revoked
              ║ 5. S = X25519(OTA_private[idx], E)            ║  ← ECDH shared secret
              ║ 6. K_s = HKDF(S, "SBOP-OTA-ENC")              ║
              ║ 7. AES-CTR decrypt with K_s → AXI SRAM         ║  ← Plaintext only in RAM
              ║ 8. zeroize(S), zeroize(K_s)                   ║
              ║ 9. Parse InnerSigBlock → sig_key_index,       ║
              ║    public_key, signature                      ║
              ║ 10. SHA-256(pubkey) == SR1.sig_fp[sig_idx]    ║  ← Signing key auth
              ║ 11. Check SR1.sig_revocation[sig_idx]         ║  ← Signing key not revoked
              ║ 12. Ed25519_verify(embedded_public_key)       ║
              ║ 13. FIH redundant verify                      ║
              ║ 14. SHA-256(payload) == header.hash            ║
              ║ 15. RE-ENCRYPT with KD → write back to NOR     ║  ← Per-device at-rest
              ╚═════════════════════════════════════════════════╝
                    │
                    ▼
              ╔══ GATE 2 (all subsequent boots) ══╗
              ║ 1. Read KD-ciphertext from NOR     ║
              ║ 2. Decrypt with KD → AXI SRAM       ║  ← Always KD, no ECDH needed
              ║ 3. Parse InnerSigBlock → key_index, ║
              ║    public_key, signature            ║
              ║ 4. SHA-256(pubkey) == SR1.fp (OTP) ║
              ║ 5. Check SR1.revocation[key_index]  ║
              ║ 6. Ed25519_verify(embedded_public_key)║
              ║ 7. SHA-256(payload) == header.hash   ║
              ║ 8. Version check + anti-rollback     ║
              ║ 9. Two-pass IWDG stop → EXECUTE      ║  ← RTC magic + NVIC_SystemReset
              ╚══════════════════════════════════════╝
```

**OTA key lifetime:**
```
OTA_KEY_PAIR[0..3] = 4 X25519 key pairs (same for all devices)
    ↑                    ↑
  OTA_private[i]      OTA_public[i] = OTA_private[i] * G
  Bootloader .rodata   Backend HSM
  (PC-ROP Zone 1)      SHA-256(OTA_public[i]) in SR1 OTP[192..319] (4 × 32 B)
                       Revocation bitmap in SR1 OTP (one-way clear)

  At boot: device derives OTA_public[i] = X25519(OTA_private[i], G)
           verifies SHA-256(OTA_public[i]) == SR1 OTP fingerprint
           checks revocation bitmap
           ECDH: S = X25519(OTA_private[i], E)
           K_s = HKDF-SHA-256(S, "SBOP-OTA-ENC")
           S and K_s are zeroized after decrypt

KD = permanent device key (unique per device)
     Always available (secure element / SVC)
     Used for at-rest encryption of all slots after first boot
     Never leaves secure element
```

**Why fixed OTA key pairs (not per-device derived):** Per-device key derivation required the backend to look up and derive a unique key for each device — N encryption operations per firmware release. With 4 fixed X25519 OTA key pairs, the backend ECDH-encrypts once and serves the same ciphertext to all devices. Per-device at-rest encryption is provided by KD re-encrypt on first boot (Stage 2). Extracting OTA_private[i] from a device enables decryption only — cannot create valid firmware without the backend's OTA_public[i] (for ECDH) and KI_private (for Ed25519 signature). The 4-key rotation with OTP revocation provides key compromise recovery.

#### 3.10.5 Why Two Gates (Revised)

| Property | Gate 1 (OTA Library) | Gate 2 (Bootloader) |
|---|---|---|
| **When** | After download, before marking VERIFIED | At every boot, before execution |
| **Decrypts?** | **No** — verifies encrypted blob hash + manifest signatures | **Yes** — ECDH with OTA_private[i] + E → K_s → decrypts → verifies → re-encrypts with KD → writes back to NOR |
| **Catches** | MITM tampering, corrupted download, wrong board/version, backend compromise | Flash corruption, physical tampering, glitch attacks, key extraction |
| **Code location** | Zone 2 application (libsbop-ota) | Zone 1 bootloader (PC-ROP secured) |
| **Failure action** | Erase slot, retry OTA | FAILSAFE, Y-Modem recovery |
| **Plaintext in NOR?** | **Never** | **Never** (decrypts directly to AXI SRAM) |
| **FIH protection** | No | Yes (redundant verify + compare) |

#### 3.10.6 Security Properties

**Power-off attack mitigation:**

```
Before (old design):
  OTA downloads → decrypts → writes plaintext to NOR → marks VERIFIED
  Window: from OTA completion to next boot
  Attacker: cuts power, removes NOR flash, reads plaintext firmware ✓

After (new design):
  OTA downloads → verifies encrypted blob → writes ciphertext to NOR → marks VERIFIED
  NOR flash: ImageHeader (plaintext, 80 B) + AES-CTR(Payload) + InnerSignatureBlock
  Window: NONE — plaintext never in NOR
  Boot: decrypts directly to AXI SRAM (volatile) → verifies in place → two-pass IWDG stop → jumps to app in AXI SRAM
  Power loss: AXI SRAM loses content → NOR only has ciphertext
  Attacker: reads NOR → gets AES-256-CTR ciphertext → cannot decrypt without OTA_private[i] and E
```

**Why the per-request response signature is acceptable:**

| Aspect | Manifest (KI_private) | Response (BACKEND_AUTH) |
|--------|----------------------|------------------------|
| Signs | Firmware identity (version, hash, board) | Per-request metadata (URL, encrypted hash, expiry) |
| Frequency | Once per firmware version | Once per device per update check |
| Key location | HSM (air-gapped) | Online backend service |
| Compromise impact | Attacker can authorize malicious firmware | Attacker can redirect download URL |

If BACKEND_AUTH is compromised, the attacker can redirect the download URL but cannot forge the manifest signature (KI_private). Gate 2 catches any malicious firmware at boot. The BACKEND_AUTH key should still be rotated on compromise per incident response.

#### 3.10.7 NOR Flash Content (per slot)

```
First boot after OTA (K_s-encrypted via X25519 ECDH, FLAG_OTA_PENDING set):
┌────────────────────────────────────────────────────────────┐
│ Offset    Size    Content                          Plaintext? │
│────────  ──────  ───────────────────────────────  ──────────│
│ 0x000000  80 B    ImageHeader                     YES       │
│                   flags & FLAG_OTA_PENDING = 1    (metadata) │
│                   enc_key_index (offset 52)       (metadata) │
│                   header_hmac (offset 56, 16 B)   (integrity)│
│ 0x000050  32 B    E (ephemeral public key)        YES       │
│                   X25519 point, sent from backend (public)   │
│ 0x000070  16 B    IV_ota (random)                 YES       │
│ 0x000080  ≤512 KB AES-256-CTR(K_s, IV_ota,        NO        │
│                   Payload)                        (ciphertext)│
│         K_s = HKDF-SHA-256(X25519(OTA_private[enc_key_index], E), "SBOP-OTA-ENC") │
│ 0x080080  116 B   InnerSignatureBlock             YES       │
│                   (outside encryption envelope)    (public   │
│                                                    signature)│
│ 0x0800F4  ~512 KB Padding / Reserved                         │
└────────────────────────────────────────────────────────────┘

Bootloader Phase 6b re-encrypts with KD → writes back
(drops E and IV_ota, replaced with single IV_dev):

All subsequent boots (KD-encrypted, FLAG_OTA_PENDING = 0):
┌────────────────────────────────────────────────────────────┐
│ Offset    Size    Content                          Plaintext? │
│────────  ──────  ───────────────────────────────  ──────────│
│ 0x000000  80 B    ImageHeader                     YES       │
│                   flags & FLAG_OTA_PENDING = 0    (metadata) │
│                   header_hmac (offset 56)         (integrity)│
│ 0x000050  16 B    IV_dev (random, new)            YES       │
│ 0x000060  ≤512 KB AES-256-CTR(KD, IV_dev,         NO        │
│                   Payload)                        (ciphertext)│
│ 0x080060  116 B   InnerSignatureBlock             YES       │
│                   (outside encryption envelope)    (public   │
│                                                    signature)│
│ 0x0800D4  ~512 KB Padding / Reserved                         │
└────────────────────────────────────────────────────────────┘

Plaintext in NOR at rest (first boot): 80 + 32 + 16 + 116 = 244 bytes (metadata + HMAC + E + IV + signature)
Plaintext in NOR at rest (subsequent): 80 + 16 + 116 = 212 bytes (metadata + HMAC + IV + signature)
Payload in NOR at rest: ALWAYS ciphertext (AES-256-CTR with K_s or KD)
K_s in NOR at rest: NEVER (ephemeral session key, zeroized after decrypt)
Plaintext Payload exists: only in AXI SRAM during execution (volatile, lost on power-off)
ImageHeader expanded from 64 B to 80 B (+16 B for HMAC field)
```

#### 3.10.8 Key Summary

**Image signing keys (NXP CSM-style multi-key, 4 slots with revocation):**

The bootloader does NOT bake public keys into .rodata (except in single-key legacy mode). Instead, each firmware image carries the public key and key_index in its SignatureBlock. The bootloader verifies SHA-256(public_key) against an OTP fingerprint stored in NOR SR1 during provisioning. Keys are revoked by clearing bits in the SR1 revocation bitmap — a one-way OTP write.

| Key | Holder | Purpose | Revocable |
|------|--------|---------|-----------|
| **KI_private[0..3]** (Ed25519 × 4) | Backend HSM (air-gapped) | Signs firmware images. Backend selects which key to use. | Yes — clear bit in SR1 key_revocation |
| **KI_public[0..3]** (Ed25519 × 4) | Embedded in image SignatureBlock | Verified against OTP fingerprint before use | Yes — by fingerprint slot |
| **BACKEND_AUTH_private** (Ed25519) | Backend online service | Signs BackendUpdateResponse (per-request) | N/A |
| **BACKEND_AUTH_public** (Ed25519) | Every device (libsbop-ota) | Verifies backend response authenticity | N/A |
| **KR** (root key) | Backend HSM only | Derives KD = HKDF(KR, composite UID) | N/A |
| **KD** (device key) | Device secure element / SVC | Derives KD_Auth, KD_Debug. Decrypts at-rest images. | N/A |
| **OTA_private[0..3]** (X25519 × 4) | Bootloader .rodata (PC-ROP) | ECDH decrypt at first boot. Extracting enables decryption only. | Yes — OTA revocation bitmap |
| **OTA_public[0..3]** (X25519 × 4) | Backend HSM | ECDH encrypt firmware once. SHA-256 fingerprint in SR1 OTP. | Yes — by fingerprint slot |
| **K_s** (AES-256, ephemeral) | CRYP register (never RAM) | AES-256-CTR decrypt at first boot. Derived via ECDH + HKDF. Zeroized after use. | N/A |

**Why 4 OTA key pairs:** Matches the signing key rotation model. Extracting OTA_private[i] from a device enables decryption only — attacker cannot create valid firmware without OTA_public[i] (for ECDH) and KI_private (for Ed25519). The 4-key rotation with OTP revocation provides key compromise recovery without bootloader update. SHA-256(OTA_public[i]) fingerprints are verified against SR1 OTP at boot.

#### 3.10.9 Key Provisioning Flow

During factory provisioning, the provisioning station burns key fingerprints into NOR SR1 (OTP). This is a one-time operation — once SR1 is OTP-locked, fingerprints cannot be modified.

```
Factory provisioning (one-time, per device):

1. Provisioning station generates or loads 4 Ed25519 key pairs:
     KI_private[0..3], KI_public[0..3]
     (All 4 are generated at the same time on the backend HSM)

2. For each key slot i in {0, 1, 2, 3}:
     fingerprint[i] = SHA-256(KI_public[i])   // 32-byte fingerprint
     nor_spi_write_sr1(KEY_FINGERPRINT_BASE + i * 32, fingerprint[i], 32)

3. Set revocation bitmap to 0x000F (all 4 keys active):
     nor_spi_write_sr1(KEY_REVOCATION_OFFSET, 0x000F, 2)

4. OTP-lock SR1 (irreversible):
     nor_spi_otp_lock_sr1()

5. Verify (read-back):
     for i in 0..4:
         assert(SR1[KEY_FINGERPRINT_BASE + i*32 .. +32] == fingerprint[i])
     assert(SR1[KEY_REVOCATION_OFFSET .. +2] == 0x000F)
     assert(SR1_OTP_LOCKED == true)

SR1 layout after provisioning:
  [0]    version_counter   : u32  (0x00000001, initial factory version)
  [4]    version_counter2  : u32  (redundant)
  [8]    version_counter3  : u32  (redundant)
  [12]   version_counter4  : u32  (redundant)
  [16]   boot_slot_selector: u8
  [17]   boot_flags        : u8   (bit 0 = key_revocation_enabled = 1)
  [18]   min_allowed_version: u32
  [22]   key_revocation    : u16  = 0x000F (all 4 slots active)
  [24]   reserved          : u8[40]
  [64]   key_fingerprint[0]: u8[32] = SHA-256(KI_public[0])
  [96]   key_fingerprint[1]: u8[32] = SHA-256(KI_public[1])
  [128]  key_fingerprint[2]: u8[32] = SHA-256(KI_public[2])
  [160]  key_fingerprint[3]: u8[32] = SHA-256(KI_public[3])
  [192..255] reserved      : u8[64]
```

**Cross-vendor compatibility:** SR1 on BY25Q32ES is 1024 bytes; on W25Q32JV is 256 bytes. All SBOP SR1 fields fit within the first 256 bytes, so both vendors are supported. Once OTP-locked, no further writes are possible.

**Device tracking:** The backend records which key fingerprints were provisioned on each device (keyed by composite UID). If a key is later compromised, the backend knows exactly which devices need revocation.

#### 3.10.10 Key Revocation Procedure

When a signing key is compromised, the backend revokes it by instructing devices to clear the corresponding bit in SR1 `key_revocation`. This is a one-way OTP write — the bit can go from 1→0 but never 0→1.

```
Revocation flow:

1. Backend detects compromise (e.g., KI_private[0] leaked).

2. For each affected device, backend sends REVOKE_KEY command:
     - During next OTA check (authenticated, TLS 1.3)
     - Or via emergency broadcast (if device has push notification capability)
     - Command: { cmd: REVOKE_KEY, key_index: 0, auth: KD_Auth_HMAC }

3. Device (libsbop-ota, Zone 2):
     a. Verifies backend auth (BACKEND_AUTH signature + KD_Auth HMAC)
     b. Calls SVC to clear revocation bit (Zone 1 only can write SR1):
          svc_revoke_key(key_index)
     c. Bootloader SVC handler:
          - Read current revocation bitmap from SR1
          - Compute new bitmap: new = current & ~(1 << key_index)
          - Write new bitmap to SR1 (OTP — bits can only clear)
          - Verify read-back: assert(SR1[KEY_REVOCATION_OFFSET] & (1 << key_index) == 0)
          - Log revocation event
     d. Return status to backend

4. Backend marks device as "key revoked" in device registry.

5. Next firmware build: backend signs with KI_private[1] (non-revoked key).
   Image carries KI_public[1] in SignatureBlock, key_index = 1.

6. Next boot on device:
     - Bootloader reads key_index = 1 from InnerSignatureBlock
     - Checks revocation[1] → bit is still set → OK
     - Verifies SHA-256(KI_public[1]) == SR1.fingerprint[1] → match
     - Verifies Ed25519(KI_public[1]) signature → valid
     - Boot succeeds
```

**Revocation bitmap semantics:**

```
Bit 0 (LSB): KI_private[0] / KI_public[0] — 0 = revoked, 1 = active
Bit 1:       KI_private[1] / KI_public[1]
Bit 2:       KI_private[2] / KI_public[2]
Bit 3 (MSB): KI_private[3] / KI_public[3]
Bits 4-15:   Reserved (must be 1 after provisioning, 0 = ignored)

Initial value after provisioning: 0x000F (all 4 slots active)
After revoking slot 0:            0x000E
After revoking slots 0,1:         0x000C
All slots revoked:                0x0000 → ANY image fails → FAILSAFE
```

**Safety properties:**

- Revocation is irreversible — bits can only go 1→0 (OTP).
- Revocation survives power loss (OTP write is atomic for a single bit clear).
- If all 4 keys are revoked (bitmap = 0x0000), the device enters FAILSAFE and requires physical recovery (DAP-Link or Y-Modem with legacy key).
- The bootloader never needs to be updated for key rotation.

#### 3.10.11 Single-Key Legacy Fallback Mode

For development boards, prototypes, or deployments that don't need multi-key CSM, the bootloader supports a single-key legacy mode via `key_index = 0xFF` in the SignatureBlock.

```
Legacy mode (key_index = 0xFF):

Condition: sig_block.key_index == 0xFF

Bootloader behavior:
  1. Skip fingerprint verification (no SR1 fingerprint slot for 0xFF)
  2. Skip revocation check (no revocation bit for 0xFF)
  3. Use public key baked into bootloader .rodata (PC-ROP protected)
  4. Ed25519_verify(ImageHeader || Payload, signature, baked_in_public_key)

Enabling legacy mode:
  - Bootloader compile flag: SBOP_KEY_MODE=single-key
  - Single public key baked into .rodata at build time
  - SR1 key_revocation_enabled flag (boot_flags bit 0) = 0
  - No fingerprints burned into SR1[64..191] (all 0xFF)

When legacy mode is active:
  - All images MUST use key_index = 0xFF
  - key_index 0-3 images are rejected (fingerprint compare fails)
  - Revocation is not supported (baked-in key is permanent)

Transition from legacy to multi-key:
  - Burn key fingerprints into SR1 (must be done before OTP lock)
  - Set boot_flags bit 0 = 1 (key_revocation_enabled)
  - From this point, key_index = 0xFF is rejected
  - All future images must use key_index 0-3
  - Irreversible after OTP lock
```

**Security note:** Legacy mode bakes the public key into bootloader .rodata. If the private key is compromised, updating the bootloader requires erasing and re-flashing internal flash (dangerous on STM32H750 — a failed internal flash update bricks the device). Multi-key CSM mode (key_index 0-3) eliminates this risk by embedding keys in images and using OTP fingerprints.

---

### 3.11 Decision: Hardened Boot Integrity (Metadata, Anti-Rollback, Health Reporting)

**Decision:** Six hardening measures are layered on top of the core boot design to protect against power-loss corruption, transient faults, firmware downgrade attacks, and manufacturing defects. Each measure is independent; failure of one does not weaken the others.

---

#### 3.11.1 Tiered Metadata Journal (Critical + Frequent)

**Problem:** A single metadata journal mixing critical slot state (updated only during OTA) with per-boot fields (boot attempt count, prev_boot_phase, IV hash) causes unnecessary write amplification. Every boot must erase a 4 KB sector and rewrite the entire journal just to update a 1-byte counter. This wastes NOR endurance (~100K cycles/sector) on fields that could be append-only. Worse, power loss during a per-boot counter update can corrupt the critical slot state that was written alongside it in the same sector.

**Design:** Split metadata into two independent journals with different update frequencies and write protocols:

| Journal | Location | Sectors | Update Trigger | Write Protocol | Record Size |
|---------|----------|---------|----------------|----------------|-------------|
| **Critical** | 0x20_0000 + 0x20_1000 | 2× 4 KB | OTA / slot transition only | Two-copy atomic (erase + write + verify + commit) | 128 B |
| **Frequent** | 0x20_2000 | 1× 4 KB | Every successful boot | Ring buffer append (no erase until wrap) | 32 B |

This eliminates ~95% of metadata sector erases. A device updated weekly with daily reboots erases its critical sectors ~52 times/year instead of ~365 times/year. The frequent journal sector handles ~128 records before one erase — at one boot/day, that's one erase every ~4 months.

**Critical Journal (two-copy atomic, NOR 0x20_0000 + 0x20_1000):**

Contains ONLY fields that change during OTA or slot state transitions. Never written during a normal boot. Written when: OTA writes a new image, slot status changes, boot_set_confirmed() promotes TESTING→ACTIVE, slot fallback occurs.

```
Copy A (0x20_0000, 4 KB sector):            Copy B (0x20_1000, 4 KB sector):
┌────────────────────────────┐              ┌────────────────────────────┐
│ [0]   sequence       : u32 │              │ [0]   sequence       : u32 │
│ [4]   active_slot    : u8  │              │ [4]   active_slot    : u8  │
│ [5]   boot_target    : u8  │              │ [5]   boot_target    : u8  │
│ [6]   re_encrypt_state:u8  │              │ [6]   re_encrypt_state:u8  │
│ [7]   reserved       : u8  │              │ [7]   reserved       : u8  │
│ [8]   slot_a_info    : 52 B│              │ [8]   slot_a_info    : 52 B│
│ [60]  slot_b_info    : 52 B│              │ [60]  slot_b_info    : 52 B│
│ [112] max_allowed_ver: u32 │              │ [112] max_allowed_ver: u32 │
│ [116] reserved2      : u8[8]│             │ [116] reserved2      : u8[8]│
│ [124] crc32          : u32 │              │ [124] crc32          : u32 │
└────────────────────────────┘              └────────────────────────────┘
  128 bytes total                             128 bytes total
  Rest of 4 KB sector unused                  Rest of 4 KB sector unused
```

**Critical journal write protocol** — same two-copy atomic sequence as before, but now only 128 bytes (vs 256), and only triggered on OTA/slot events:

```
function critical_journal_write(new_data):
    let (active_idx, inactive_idx, current_seq) = critical_journal_find_active()
    new_seq = current_seq + 1  // odd = writing
    new_data[0..4] = new_seq
    new_data[124..128] = crc32(new_data[0..124])
    nor_spi_erase_sector(inactive_sector_base)        // Separate 4 KB sector
    nor_spi_page_program(inactive_sector_base, new_data, 128)
    verify_data = nor_spi_read(inactive_sector_base, 128)
    assert(crc32(verify_data[0..124]) == verify_data[124..128])
    new_seq = current_seq + 2  // even = committed
    nor_spi_write_u32(inactive_sector_base, new_seq)  // Atomic 4-byte commit
```

**Frequent Journal (ring buffer, NOR 0x20_2000, 4 KB sector):**

Records per-boot information. Simple append-only ring buffer — no sector erase needed until the ring wraps (~128 records). A CRC-8 on each record detects partial writes from power loss. Scanning backward from the end of the sector finds the most recent valid record.

```
Frequent record (32 bytes each):
┌────────────────────────────┐
│ [0]  boot_seq        : u32 │  Monotonic boot counter (1, 2, 3, ...)
│ [4]  prev_boot_phase : u8  │  0x00 = did not reach EXECUTE
│     .                        │  0x01 = reached EXECUTE (normal boot)
│     .                        │  0x02 = confirmed (app called boot_set_confirmed)
│ [5]  flags           : u8  │  bit 0: was_testing on this boot
│     .                        │  bit 1: was_fallback slot
│     .                        │  bits 2-7: reserved
│ [6]  reserved         : u8[2]│
│ [8]  last_iv_hash     : u8[24]│ Truncated SHA-256(IV_dev || KD[0..16])
│ [32] (end)                   │
└────────────────────────────┘
```

**Frequent journal operations:**

```
function frequent_journal_append(phase, flags, iv_hash):
    let last = frequent_journal_read_last()
    let new_seq = (last.valid) ? last.boot_seq + 1 : 1
    let write_off = frequent_journal_find_next_free()
    if write_off >= 4096 - 32:
        nor_spi_erase_sector(0x20_2000)   // Ring wrapped — erase and restart
        write_off = 0
    let record = pack(new_seq, phase, flags, iv_hash[0..24])
    nor_spi_page_program(0x20_2000 + write_off, record, 32)
    let verify = nor_spi_read(0x20_2000 + write_off, 32)
    assert(memcmp(record, verify, 32) == 0)

function frequent_journal_read_last() -> Option<FrequentRecord>:
    // Scan backward from end for first non-erased record with valid CRC-8
    for off in (4096 - 32) down to 0 step 32:
        data = nor_spi_read(0x20_2000 + off, 32)
        if data[0..4] != 0xFFFFFFFF:          // Not erased
            if crc8(data[0..31]) == data[31]: // CRC-8 check
                return Some(parse(data))
    return None  // Sector empty (all 0xFF) or all records corrupt

function frequent_journal_find_next_free() -> u32:
    for off in 0..4096 step 32:
        data = nor_spi_read(0x20_2000 + off, 4)
        if data[0..4] == 0xFFFFFFFF:
            return off
    return 4096  // Sector full
```

**Power-loss analysis for frequent journal:**

| Failure Point | What Happens | Impact |
|---------------|-------------|--------|
| Power loss during page program | Record has partial data, CRC-8 mismatch | `read_last()` skips it, finds previous record. One boot record lost — harmless. |
| Power loss during sector erase | All records lost | boot_seq resets to 1, IV reuse check is best-effort (no history to compare = no false positive) |
| Power loss between append and verify | Record may be corrupt | Same as partial write — skipped by CRC check |

In all cases, **critical slot state is never at risk** — the frequent journal is a separate sector, and the critical journal is not touched during normal boots.

**IV reuse detection with frequent journal:**

```
// Phase 6 (VERIFY_INTEGRITY): check IV reuse
let last = frequent_journal_read_last()
if last.valid:
    let expected_hash = sha256(current_iv || KD[0..16])
    if last.last_iv_hash == expected_hash[0..24]:
        // Same (KD, IV) pair used twice — possible AES-CTR keystream replay
        boot_enter_failsafe(ERR-BOOT-CRYPTO-007)

// Phase 11 (EXECUTE): record this boot's IV hash for next boot's check
let new_iv_hash = sha256(current_iv || KD[0..16])
frequent_journal_append(0x01, flags, new_iv_hash)
```

**Why last_iv_hash is truncated to 24 bytes:** The field must fit in a 32-byte record alongside boot_seq, phase, and flags. A 24-byte truncated hash retains 192 bits of collision resistance — an attacker would need ~2^96 attempts to find an IV that produces the same truncated hash, well beyond brute-force feasibility. The full SHA-256 is still computed; only the stored fingerprint is truncated.

**Migration from single journal:** At first boot after this change, the bootloader detects the old 256-byte journal format (CRC at offset 252 vs 124). It reads the old format once, writes both new journals (critical from old slot/version fields, frequent from old boot_attempt/prev_boot_phase/IV fields), then never reads the old format again. The old sectors are left intact for rollback compatibility.

**Cost:** Three 4 KB sectors (12 KB total, +4 KB vs prior), +~450 bytes code. Critical journal rarely erased (OTA events only). Frequent journal erased once per ~128 boots. Eliminates ~95% of metadata NOR erases vs single journal.

---

#### 3.11.2 NOR Hardware Write-Protect for Metadata

**Problem:** A runaway application or OTA library bug could write to the metadata area (0x20_0000–0x21_FFFF), corrupting slot tracking. Software protection (check address range before write) is only as strong as the code that enforces it.

**Design:** After `LOCK_BOOT` phase, the bootloader sets the BY25Q32ES block-protect bits to hardware-lock the metadata region before jumping to the application.

```
// Phase 10 (LOCK_BOOT) — before EXECUTE:

// BY25Q32ES status register layout:
//   CMP  SEC  TB  BP2  BP1  BP0  WEL  WIP
// With CMP=0, SEC=0, TB=0:
//   BP2..BP0 protect from top of flash downward

// Protect upper 2 MB (metadata + reserved at 0x20_0000–0x3F_FFFF):
//   BP2=1, BP1=0, BP0=0 → protect upper 1/2 = upper 2 MB
nor_spi_write_status_register(BP2_BIT | SRP1_ENABLE);

// After this, any write to 0x20_0000+ returns NOP.
// Slots (0x00_0000–0x1F_FFFF) remain writable for OTA.

// Note: OTA library unlocks via SRP before writing, re-locks after.
// Unlock requires writing status register with /WP pin held high —
//   an errant memory write won't accidentally unlock.
```

**Application unlock/re-lock (libsbop-ota):**

```
// OTA library: before updating slot metadata
nor_spi_write_enable();                     // Set WEL
nor_spi_write_status_register(0x00);        // Clear BP bits (unlock)
nor_spi_erase_sector_and_write(metadata);   // Update metadata
nor_spi_write_enable();
nor_spi_write_status_register(BP2_BIT | SRP1_ENABLE);  // Re-lock
```

**Cost:** 2 SPI command sequences, zero flash bytes. Hardware-enforced, survives application crash.

---

#### 3.11.3 Test/Confirm Model (MCUboot-Style Immediate Rollback)

**Problem:** The N-attempt policy tolerates up to N consecutive failures before rollback. Each failed boot means the device is unavailable for ~300 ms (full verification + app init → crash → reset). For a device that reboots every 30 seconds, 3 attempts = 90 seconds of unavailability. Worse, a crashing application that manages to call the health API before crashing can reset the counter and prevent rollback indefinitely.

**Design:** Adopt MCUboot's test/confirm model: after a successful update, the new image enters **TESTING** state, not ACTIVE. The application must call `boot_set_confirmed()` to promote itself to ACTIVE. If the device resets for ANY reason before confirmation, the bootloader **immediately** reverts to the fallback slot — no retry count, no N-attempt accumulation.

This is strictly better than N-attempt for three reasons:
1. **Faster recovery:** One failed boot triggers rollback, not N failed boots.
2. **No counter management:** The application doesn't need to track or clear attempt counts — a single API call confirms health.
3. **Matches industry practice:** MCUboot, Android A/B updates, and ChromeOS all use this model.

```
Slot state machine (test/confirm extension):

  PENDING ──(verify OK, first boot)──> TESTING
  TESTING ──(app calls boot_set_confirmed())──> ACTIVE
  TESTING ──(reset without confirm)──> INVALID  // Immediate rollback
  ACTIVE ──(new OTA to other slot)──> FALLBACK
```

**Bootloader logic (SELECT_SLOT phase):**

```
function boot_select_slot():
    let boot_slot = metadata_read_boot_target()
    let slot_status = storage_get_slot_status(boot_slot)

    if slot_status == TESTING:
        // Application never confirmed — image is unhealthy.
        // Immediate rollback. No retry.
        error_log_write(SELECT_SLOT, ERR-BOOT-STATE-005)
        storage_set_slot_status(boot_slot, INVALID)
        let fallback = (boot_slot == SLOT_A) ? SLOT_B : SLOT_A
        let fb_status = storage_get_slot_status(fallback)
        if fb_status == ACTIVE or fb_status == FALLBACK:
            metadata_set_boot_target(fallback)
            system_reset()  // Restart with fallback slot
        else:
            boot_enter_failsafe(ERR-BOOT-STATE-004)  // No bootable slot

    // ... continue to boot_verify_and_execute_slot(boot_slot) ...
```

**Application confirm API:**

```
/// Called by the application after self-test, sensor check, and network
/// connectivity are confirmed. Typically in early init, before the main loop.
function boot_set_confirmed() -> Result<(), BootError>:
    let active_slot = metadata_read_active_slot()
    let slot_status  = storage_get_slot_status(active_slot)

    if slot_status != TESTING:
        return Err(ERR-BOOT-STATE-006)  // Already confirmed or wrong state

    storage_set_slot_status(active_slot, ACTIVE)
    // Transition is atomic via metadata journal two-copy write
    return OK
```

**Application integration:**

```c
// main.c — early init
int main(void) {
    hal_init();
    sensors_init();
    network_init();

    // Confirm this firmware is healthy BEFORE starting the main loop.
    // If we crash before this point, the bootloader reverts to the
    // previous firmware on the next reset.
    if (self_test_pass() && sensors_calibrated() && network_link_up()) {
        boot_set_confirmed();  // TESTING → ACTIVE
    }
    // If we DON'T call boot_set_confirmed() and the device resets,
    // the bootloader immediately reverts — no second chance.

    // ... main loop ...
}
```

**Failure scenarios:**

| Scenario | Slot status before boot | Action |
|---|---|---|
| Normal boot, app confirms health | TESTING → boot_set_confirmed() → ACTIVE | Boot as ACTIVE next time |
| App hardfaults before confirming | TESTING (never confirmed) | Immediate rollback to fallback |
| Power loss during app init | TESTING (never confirmed) | Immediate rollback to fallback |
| App confirms, normal reboot | ACTIVE | Normal boot |
| App confirms, OTA updates other slot | ACTIVE → FALLBACK | Old image becomes safety net |
| Both slots TESTING/INVALID | — | Enter FAILSAFE → serial recovery |

**Comparison with N-attempt policy:**

| Metric | N-Attempt Policy | Test/Confirm Model |
|---|---|---|
| Time to recovery | N × (boot_time + app_crash_time) | 1 × boot_time |
| Unavailability window | 90 s (3 attempts × 30 s) | 0.3 s (immediate rollback) |
| Application complexity | Must manage attempt counter | Single API call |
| Transient fault tolerance | Tolerates N-1 faults | No tolerance (one crash = rollback) |
| Counter reset attack surface | App must clear counter (can be spoofed) | Bootloader manages state (can't be spoofed) |

**Transient fault mitigation:** The loss of N-attempt transient fault tolerance is addressed by application-level watchdogs and health checks. If the application can reboot cleanly but consistently fails health checks, it SHOULD be rolled back — the image has a real bug, not a transient fault. Hardware-level transients (EMI, power dip) that cause CPU faults should trigger the HardFault handler → error log → system reset, which the test/confirm model correctly treats as a failure.

**Cost:** replaces N-attempt logic (~200 bytes code, 2 bytes metadata) with test/confirm logic (~100 bytes code, 1 byte metadata). Net savings: ~100 bytes code, 1 byte metadata.

---

#### 3.11.4 Zero-Tolerance Header Validation

**Problem:** A malformed ImageHeader can confuse the parser. Any field that must have a specific value must be checked — no silent acceptance of unexpected values.

**Design:** Every header field is validated with explicit constraints. No "best effort" parsing.

```
// Phase 4 (PARSE_HEADER) — mandatory checks, all FIH-protected:
// Each check uses fih_dec() on failure, no early return

// Fixed-value fields
check(header.magic             == 0x53424F50,  ERR-BOOT-PARSE-001);  // "SBOP"
check(header.header_version    == 1,           ERR-BOOT-PARSE-002);  // Version 1 only
check(header.header_size        == 80,          ERR-BOOT-PARSE-003);  // Exactly 80 bytes

// Range-check fields
check(header.image_size        >= 1024,         ERR-BOOT-PARSE-004);  // Min 1 KB
check(header.image_size        <= 524288,       ERR-BOOT-PARSE-004);  // Max 512 KB (AXI)
check(header.image_size        % 4 == 0,        ERR-BOOT-PARSE-004);  // Word-aligned

// Board ID: must match this board's SKU
// Read board_id from Board Info Area (NOR, once), compare to header
board_id = board_info_read_board_id()
check(header.board_id           == board_id,     ERR-BOOT-PARSE-008);  // Wrong SKU
// FIH: redundant read + compare
board_id_v2 = board_info_read_board_id()
check(board_id == board_id_v2,                   ERR-BOOT-STATE-003);
check(header.board_id           == board_id_v2,  ERR-BOOT-PARSE-008);

// Version
check(header.firmware_version  > 0,            ERR-BOOT-VERSION-003); // Non-zero
check(header.firmware_version  <= 0xFFFFFFFF,   ERR-BOOT-VERSION-003); // Not rollover

// Flags: only known flags may be set
uint32_t known_flags = FLAG_ENCRYPTED | FLAG_SECOND_STAGE | FLAG_FACTORY_TEST;
check((header.flags & ~known_flags) == 0,       ERR-BOOT-PARSE-005);

// Reserved: must be zero
check(header.reserved_is_zero(),                ERR-BOOT-PARSE-005);

// Hash: cannot be all-zero (uninitialized image)
check(!is_zero(header.hash, 32),               ERR-BOOT-PARSE-007);

// SignatureBlock CRC: validate immediately after parsing
check(sig_block.crc32 == crc32(&sig_block, 96), ERR-BOOT-CRYPTO-001);

// All checks passed → fih_ret(fih_rc) == true → proceed to Phase 5
```

**Cost:** Zero additional code. These replace generic checks with specific ones. The `is_zero()` and `reserved_is_zero()` helpers are ~10 bytes each.

---

#### 3.11.5 Anti-Rollback: Redundant Version Counter with FIH Protection

**Problem (beyond current design):** The version counter in NOR SR1 is the primary anti-rollback mechanism, but:
1. The comparison `firmware_version >= stored_version` is a single instruction that a glitch could skip.
2. The redundant FIH check already reads `stored_version` twice, but a glitch on BOTH reads targeting the same SR1 address could defeat it.
3. The version counter itself can only be written once (OTP) — if a buggy OTA writes version 999999, the device is permanently locked to that version or higher.

**Design:** Three layers of anti-rollback, each independent.

**Layer 1 — OTP version counter in NOR SR1 (hardware root):**

```
SR1 layout (first 256 bytes of 256-byte usable window):
  // ── Version control (20 B) ──
  [0]  version_counter  : u32  (monotonic, OTP — can only increase)
  [4]  version_counter2 : u32  (redundant copy, 3-of-4 majority vote)
  [8]  version_counter3 : u32  (redundant copy)
  [12] version_counter4 : u32  (redundant copy)
  [16] boot_slot_selector: u8  (0=A, 1=B)
  [17] boot_flags        : u8  (bit 0 = sig_key_revocation_enabled)
  [18] min_allowed_version: u32 (minimum version allowed — factory floor)
  // ── Signing key revocation (2 B) ──
  [22] sig_key_revocation : u16 (OTP — bit[i]=0 permanently revokes signing key slot i.
                                  Writable one-way: 0xFFFF at factory → clear bits to revoke.
                                  0xFFFF = all signing keys active. 0x0000 = all revoked → legacy mode.)
  [24] reserved          : u8[40]
  // ── Signing key fingerprints (128 B) — Ed25519 KI_public[0..3] ──
  [64] sig_fingerprint[0]: u8[32]  // SHA-256 of KI_public[0] (primary signing key)
  [96] sig_fingerprint[1]: u8[32]  // SHA-256 of KI_public[1] (secondary signing key)
  [128] sig_fingerprint[2]: u8[32] // SHA-256 of KI_public[2] (backup signing key)
  [160] sig_fingerprint[3]: u8[32] // SHA-256 of KI_public[3] (backup signing key)
  [192..255] reserved    : u8[64]  (for future use; all SR offsets ≤ 0xFF for cross-vendor compat)

SR2 layout (first 256 bytes of 256-byte usable window):
  // ── OTA encryption key revocation (2 B) ──
  [0]  ota_key_revocation : u16 (OTP — bit[i]=0 permanently revokes OTA key pair i.
                                  Writable one-way: 0xFFFF at factory → clear bits to revoke.
                                  0xFFFF = all OTA keys active.)
  [2]  reserved           : u8[62]
  // ── OTA encryption key fingerprints (128 B) — X25519 OTA_public[0..3] ──
  [64] ota_fingerprint[0] : u8[32] // SHA-256 of OTA_public[0] (primary OTA key)
  [96] ota_fingerprint[1] : u8[32] // SHA-256 of OTA_public[1] (secondary OTA key)
  [128] ota_fingerprint[2]: u8[32] // SHA-256 of OTA_public[2] (backup OTA key)
  [160] ota_fingerprint[3]: u8[32] // SHA-256 of OTA_public[3] (backup OTA key)
  [192..255] reserved     : u8[64] (for future use)

Rationale for SR1/SR2 split:
  - SR1: Signing keys + version control — needed by ALL boots (Zone 1 bootloader, every boot)
  - SR2: OTA encryption keys — needed only on first boot after OTA (ECDH decrypt path)
  - Separate security registers prevent a single SPI command glitch from
    affecting both signing and encryption verification paths
  - Both registers use cross-vendor compatibility pattern (offsets ≤ 0xFF)

Version counter write protocol:
  1. Write new version to counter (increment by exactly 1)
  2. Verify read-back
  3. Write redundant copies 2, 3, 4
  4. Verify at least three of four match the written value
  5. If < 3 match → FAILSAFE (SR1 corruption beyond single-bit recovery)

Version counter read + verify (FIH-protected, 3-of-4 majority vote):
  v1 = nor_spi_read_security_register(1, offset=0, len=4)
  v2 = nor_spi_read_security_register(1, offset=4, len=4)
  v3 = nor_spi_read_security_register(1, offset=8, len=4)
  v4 = nor_spi_read_security_register(1, offset=12, len=4)

  // Count occurrences of each distinct value
  // Accept any value that appears in 3+ of the 4 copies
  // This tolerates a single-bit corruption in one copy without bricking the device
  if v1 == v2 and v1 == v3:     return v1   // v1 matches 3 copies
  if v1 == v2 and v1 == v4:     return v1   // v1 matches 3 copies
  if v1 == v3 and v1 == v4:     return v1   // v1 matches 3 copies
  if v2 == v3 and v2 == v4:     return v2   // v2 matches 3 copies
  // No value has majority (2-2 split or worse) — SR1 is corrupted
  return VERSION_ERR  // → FAILSAFE
```

**Layer 2 — Minimum allowed version (factory floor):**

```
SR1 field: min_allowed_version : u32

At factory provisioning:
  min_allowed_version = factory_firmware_version
  // Stored in SR1 alongside version_counter

At boot:
  check(header.firmware_version >= min_allowed_version, ERR-BOOT-VERSION-004)

Purpose:
  - Prevents booting ancient pre-production firmware images
  - Even if an attacker somehow resets the version counter,
    they can't boot firmware older than the factory floor
  - Stored in OTP — can never be lowered
```

**Layer 2b — Maximum allowed version (ceiling, metadata-backed):**

The version counter in SR1 OTP can only increment by 1 per boot. However, an attacker who compromises the signing key can still burn through the counter one version at a time. A ceiling prevents this by capping the maximum version the bootloader will accept. The ceiling is stored in the metadata journal (not OTP) because it must be field-updatable — the backend increments it as part of authorized OTA updates.

```
Metadata field: max_allowed_version : u32

At factory provisioning:
  metadata.max_allowed_version = factory_firmware_version

During authorized OTA (backend pushes new ceiling):
  // BackendUpdateResponse includes new_max_allowed_version
  // Signed by BACKEND_AUTH key, verified in Gate 1
  if update_info.new_max_allowed_version > metadata.max_allowed_version:
      metadata.max_allowed_version = update_info.new_max_allowed_version

At boot:
  check(header.firmware_version <= metadata.max_allowed_version,
        ERR-BOOT-VERSION-006)  // Version exceeds ceiling

Purpose:
  - Prevents attacker with compromised signing key from burning
    the OTP version counter past the authorized ceiling
  - Backend controls the version window: [floor, ceiling]
  - Even if signing key is compromised, attacker cannot install
    firmware with version > ceiling
  - Ceiling can be raised by backend (signed command), never lowered
  - Stored in metadata journal (two-copy atomic) — survives power loss
```

**Anti-rollback version window:**

```
  Version number space:
  
  0 ─── min_allowed_version ─── stored_version ─── max_allowed_version ─── 0xFFFFFFFF
         (SR1 OTP floor)         (SR1 OTP counter)   (metadata ceiling)
         
         ├──── REJECT ────┼────── ALLOW ──────┼────────── REJECT ──────────┤
         │  Too old       │  Valid range      │  Exceeds ceiling            │
         │  (factory      │  (normal          │  (compromised signing       │
         │   floor)       │   operation)      │   key or attack)            │
```

**Layer 3 — OTA pre-check (defense in depth):**

```
// libsbop-ota: ota_check_for_update()
// Before downloading, verify backend is offering a version >= current

function ota_check_for_update(backend_url, current_version, result):
    update_info = backend_request_update(device_id, current_version, auth_token)

    if update_info.firmware_version < current_version:
        return ERR-OTA-VERSION-001  // Backend offered a downgrade — reject

    // Backend signature covers version — verified separately in Gate 1
    result = update_info
    return OK
```

**Full anti-rollback chain at boot:**

```
Phase 7 (CHECK_VERSION):
  // FIH accumulator — all checks contribute
  fih_int fih_rc = FIH_SUCCESS;

  // Layer 1: OTP version counter (4-way redundant)
  stored_version = version_counter_read_with_redundancy()
  check(stored_version != VERSION_ERR,        ERR-BOOT-VERSION-002)

  // Layer 1a: Redundant read from different SR1 offsets (glitch defense)
  stored_version_v2 = version_counter_read_with_redundancy()
  check(stored_version == stored_version_v2,  ERR-BOOT-STATE-003)

  // Layer 2: Minimum allowed version (factory floor)
  min_version = nor_spi_read_security_register(1, offset=18, len=4)
  check(header.firmware_version >= min_version, ERR-BOOT-VERSION-004)

  // Layer 2a: Redundant min version read
  min_version_v2 = nor_spi_read_security_register(1, offset=18, len=4)
  check(min_version == min_version_v2,        ERR-BOOT-STATE-003)

  // Layer 2b: Maximum allowed version (ceiling, metadata-backed)
  max_version = metadata_read_max_allowed_version()
  check(header.firmware_version <= max_version, ERR-BOOT-VERSION-006)
  // Redundant ceiling read
  max_version_v2 = metadata_read_max_allowed_version()
  check(max_version == max_version_v2,        ERR-BOOT-STATE-003)

  // Core check: firmware >= stored
  check(header.firmware_version >= stored_version, ERR-BOOT-VERSION-001)

  // Redundant: same check, second execution (FIH)
  stored_version_v3 = version_counter_read_with_redundancy()
  check(header.firmware_version >= stored_version_v3, ERR-BOOT-VERSION-001)

  // All passed
  check(fih_ret(fih_rc) == true, ERR-BOOT-VERSION-001)
```

**Version counter increment protocol (COMMIT_VERSION phase):**

```
Phase 8 (COMMIT_VERSION):
  if header.firmware_version == stored_version:
      // Same version — no counter increment needed (OTA retry, slot swap)
      proceed
  elif header.firmware_version > stored_version:
      // New version — increment counter by exactly 1
      // Guard: never increment by more than 1 per boot
      check(header.firmware_version == stored_version + 1,
            ERR-BOOT-VERSION-005)  // Version skip detected

      new_version = stored_version + 1
      version_counter_write_with_redundancy(new_version)
      // Verify all four copies written correctly
      verify_version = version_counter_read_with_redundancy()
      check(verify_version == new_version, ERR-STOR-WRITE-001)
  else:
      // firmware_version < stored_version — already caught in Phase 7
      boot_enter_failsafe(ERR-BOOT-VERSION-001)
```

**Anti-rollback attack resistance:**

| Attack | Defense |
|---|---|
| Swap NOR flash to older version | Version counter in SR1 (OTP) — can only increase, so old version < stored → rejected |
| Glitch skip version comparison | FIH: 6 independent version comparisons, all must pass |
| Glitch skip SR1 read | 4 redundant copies, all compared, FIH double-read |
| Replay old provisioning data | min_allowed_version in SR1 (OTP) — old firmware < factory floor → rejected |
| Backend offers downgrade | OTA library checks version ≥ current before downloading |
| Buggy OTA writes inflated version | Increment capping: version can only increase by 1 per boot. A jump from 5→999999 is rejected with ERR-BOOT-VERSION-005 |

**Cost:** Already within existing SR1 allocation (OTP), +~200 bytes code for Layer 2 and the redundant read protocol. The 4-way redundant counter and min_allowed_version use bytes already reserved in SR1.

---

#### 3.11.6 Manufacturing End-of-Line Power-Cycle Test

**Problem:** A provisioning script error (wrong bootloader, wrong application, missed SR lock) produces a device that passes DAP-Link verification but fails to boot. These are discovered in the field.

**Design:** The Python provisioning script performs a power-cycle test after programming Option Bytes.

```
EOL test (Python + pyOCD, after Pass 3 provisioning):

  # 1. Reset MCU (or toggle relay for full power cycle)
  pyocd.reset()

  # 2. Wait for boot to complete (bootloader + app init)
  time.sleep(1.0)  # Bootloader < 300 ms, app init < 700 ms

  # 3. Read boot semaphore from AXI SRAM
  #    Bootloader writes 0x424F4F54 at 0x2400_0000 before EXECUTE
  #    Application writes 0x41505000 at 0x2400_0004 at main()
  boot_magic = pyocd.read32(0x2400_0000)
  app_magic  = pyocd.read32(0x2400_0004)

  if boot_magic != 0x424F4F54:  # "BOOT"
      FAIL("Bootloader did not reach EXECUTE phase")

  if app_magic != 0x41505000:   # "APP\x00"
      FAIL("Application did not start — check image validity")

  # 4. Read boot metrics
  boot_count = pyocd.read_sr(1, offset=0)  # Version counter
  slot_a_status = pyocd.read32(0x20_0008)  # slot_a_info[0] = status
  if slot_a_status != STATUS_ACTIVE:
      FAIL("Slot A not marked ACTIVE after factory provisioning")

  # 5. Read error log (should be empty)
  error_log_head = pyocd.read32(0x20_0080)
  if error_log_head != 0:
      FAIL("Boot error logged — check error code")

  # 6. PASS
  log(f"DUT {device_id}: EOL test PASS. Version={boot_count}, Status=OK")
```

**Boot semaphore (in bootloader):**

```
// Phase 11 (EXECUTE), before jumping:
*(volatile uint32_t *)0x2400_0000 = 0x424F4F54;  // "BOOT"
// ... copy image, set VTOR, clear state ...
// Jump to application
```

**Application semaphore:**

```
// Application main(), first thing:
int main(void) {
    *(volatile uint32_t *)0x2400_0004 = 0x41505000;  // "APP\0"

    // Confirm boot health — promotes slot from TESTING to ACTIVE.
    // Must be called once after self-tests pass. See §3.11.3 (test/confirm model).
    // The bootloader provides boot_set_confirmed() as a weak symbol.
    if (self_test_pass() && sensors_ok()) {
        boot_set_confirmed();
    }
    // ... rest of application init ...
}
```

**Cost:** +8 bytes AXI SRAM, +2 instructions in bootloader, +1 instruction in application.

---

#### 3.11.7 Compiler Stack Protection

**Problem:** Stack buffer overflows in the bootloader (e.g., malformed image header parsing, Y-Modem frame handling, string operations on attacker-controlled data) can overwrite return addresses and hijack control flow. The bootloader handles attacker-controlled input (NOR flash contents, UART recovery data) and runs before any OS-level protections.

**Design:** Enable `-fstack-protector-strong` in the bootloader compilation to insert stack canaries on functions with local arrays, structs with arrays, or address-taken locals.

```
# Bootloader compilation flags (GCC / arm-none-eabi-gcc):
CFLAGS  += -fstack-protector-strong
CFLAGS  += --param=ssp-buffer-size=4   # Protect buffers >= 4 bytes

# Linker: ensure __stack_chk_guard and __stack_chk_fail are in bootloader .text
# (not pulled from libc, which may not exist in bare-metal)
```

**Stack canary implementation (bare-metal):**

```c
// In bootloader start-up, before any C function with stack arrays:
#include <stdint.h>

// Random canary value — TRNG read at INIT phase
uintptr_t __stack_chk_guard;

void bootloader_init_canary(void) {
    // Use TRNG (hardware, available at INIT) for the canary
    // Fallback: use a constant from device unique ID if TRNG fails
    if (HAL_RNG_GenerateRandomNumber(&hrng, &__stack_chk_guard) != HAL_OK) {
        // Derive from unique device properties (not a compile-time constant)
        __stack_chk_guard = *(uint32_t *)0x1FF1E800   // MCU UID word 0
                          ^ *(uint32_t *)0x1FF1E804;  // MCU UID word 1
    }
}

// Called on stack smash detection — never returns
__attribute__((noreturn)) void __stack_chk_fail(void) {
    // Log the event and enter FAILSAFE immediately
    // No printf, no complex logic — minimal attack surface
    error_log_write(FAILSAFE, ERR-BOOT-STATE-005);
    crypto_hw_key_zeroize();
    secure_zeroize_boot_workspace();
    platform_signal_failsafe(ERR-BOOT-STATE-005.code);
    while (1) { __WFI(); }
}
```

**Coverage of `-fstack-protector-strong`:**

| Protected | Not protected |
|-----------|---------------|
| Functions with local char/int arrays >= 4 bytes | Functions with only scalar locals |
| Functions with structs containing arrays | Functions with no local stack usage |
| Functions with address-taken locals | Functions GCC determines have no risk |
| All Phase handlers (each has local arrays/buffers) | Trivial accessors |

**Trade-offs:**

| Aspect | Impact |
|--------|--------|
| Code size | +~400 bytes (canary setup + check per protected function) |
| Boot time | +~0.5 ms (TRNG read for canary init, ~20 cycles per function check) |
| Stack per function | +4 bytes (canary slot) |
| Detection coverage | Strong against linear stack buffer overflows; no protection against arbitrary write or frame pointer overwrite |

**Why not `-fstack-protector-all`:** All-function protection adds ~1.2 KB code and ~0.5 ms boot time for marginal gain — small functions without arrays are unlikely overflow targets.

**Cost:** +~400 bytes flash, +0.5 ms boot time, +4 bytes stack per protected function.

---

**Hardening summary:**

| # | Measure | Flash | Code | Prevents |
|---|---|---|---|---|
| 3.11.1 | Tiered metadata journal (critical + frequent) | +12 KB (3 sectors) | +450 B | Power-loss slot corruption, NOR wear from per-boot erases |
| 3.11.2 | NOR BP write-protect metadata | 0 | 0 B | Runaway write to metadata area |
| 3.11.3 | Test/confirm model (immediate rollback) | 0 | +150 B | Multi-boot unavailability window from N-attempt |
| 3.11.4 | Zero-tolerance header validation | 0 | 0 B | Malformed header parser confusion |
| 3.11.5 | Anti-rollback (4-way redundant + min version + capped increment) | 0* | +200 B | Glitch bypass of version check, version inflation |
| 3.11.6 | Manufacturing EOL power-cycle test | +8 B** | +2 insns | Undetected provisioning failures |
| 3.11.7 | Compiler stack protection (-fstack-protector-strong) | +400 B | +0.5 ms | Stack buffer overflow → control-flow hijack |
| 3.12   | Automatic slot fallback | 0 | +350 B | Single slot failure → permanent FAILSAFE |
| 3.13   | IWDG hardware watchdog (800 ms, two-pass) | 0 | +200 B | Bootloader hang → unrecoverable |
| **Total** | | **+12,667 B** | **+2,050 B** | |

\* Uses bytes already allocated in NOR SR1. \** AXI SRAM, not flash.

Total bootloader budget impact: ~2,050 bytes additional code, well within the ~57 KB reserved space. Updated bootloader .text: 37 KB → ~39.0 KB. The 12 KB metadata area (three 4 KB sectors) fits within the 128 KB allocated metadata region at 0x20_0000.

---

### 3.12 Decision: Automatic Slot Fallback Before FAILSAFE

**Decision:** The bootloader must attempt to boot the fallback slot before entering FAILSAFE. The main boot flow is modified to implement a two-slot retry loop: try the selected slot, if it fails, try the other slot, only enter FAILSAFE if both are dead.

**Problem:** The current boot flow enters FAILSAFE on the first slot verification failure. If Slot A has a transient NOR read error (bit flip, EMI), the device enters FAILSAFE even though Slot B has a perfectly valid image. Similarly, power loss during Phase 6b re-encrypt corrupts one slot — but the other is still valid (§3.10.7). The device should automatically fall back rather than requiring Y-Modem recovery.

**Design — Modified boot flow with slot retry:**

```
function sbop_boot() -> Never:
    // ── Phase 1: Hardware Initialization (INIT) ──
    state = INIT
    assert_eq(platform_init(), OK)

    // Check tamper state before anything else
    let tamper_state = tamper_detect_get_state()
    if tamper_state == TAMPERED or tamper_state == COMPROMISED:
        handle_tamper_failure(tamper_state)
        // Does not return

    // ── Phase 2: Select Boot Slot ──
    state = SELECT_SLOT
    let boot_slot = storage_get_boot_slot()
    assert(boot_slot == SLOT_A or boot_slot == SLOT_B, ERR-BOOT-STATE-001)

    // ── Phase 3-9: Attempt to verify and boot the selected slot ──
    let result = boot_verify_and_execute_slot(boot_slot)

    if result == OK:
        // Never returns — jumped to application
        unreachable

    // ── Selected slot failed — try the fallback slot ──
    // IWDG is active (Pass 1, 800 ms timeout). Each NOR operation below
    // may take 10-45 ms. Refresh before and after the fallback sequence.
    IWDG_REFRESH()
    error_log_write(state, result.error)
    IWDG_REFRESH()
    storage_set_slot_status(boot_slot, INVALID)

    let fallback_slot = (boot_slot == SLOT_A) ? SLOT_B : SLOT_A
    let fallback_status = storage_get_slot_status(fallback_slot)

    if fallback_status == ACTIVE or fallback_status == FALLBACK or fallback_status == VERIFIED:
        // Fallback slot has a valid image — try it.
        // VERIFIED means Gate 1 passed (OTA downloaded + backend signature OK)
        // but Gate 2 hasn't run yet — still a valid candidate for fallback.
        state = SELECT_SLOT
        IWDG_REFRESH()
        storage_set_boot_slot(fallback_slot)
        IWDG_REFRESH()
        metadata_set_boot_attempt_count(0)
        metadata_set_prev_boot_phase(0x00)

        let fallback_result = boot_verify_and_execute_slot(fallback_slot)
        if fallback_result == OK:
            // Never returns — jumped to fallback application
            unreachable

        // Both slots failed — enter FAILSAFE
        IWDG_REFRESH()
        storage_set_slot_status(fallback_slot, INVALID)

    // ── Both slots dead — enter FAILSAFE ──
    boot_enter_failsafe(ERR-BOOT-STATE-004)  // No bootable slot
```

**`boot_verify_and_execute_slot()` — single-slot verification (Phases 3-11):**

```
function boot_verify_and_execute_slot(slot: SlotID) -> Result<Never, BootError>:

    // ── Phase 3: Load Image ──
    state = LOAD_IMAGE

    // Recovery: check for interrupted re-encrypt (write-before-erase recovery)
    let re_state = metadata_get_re_encrypt_state()
    if re_state == RE_STATE_STAGED:
        // Power was lost between staging write and commit.
        // Staging area at slot_base + 0x8_0000 has verified KD-encrypted data.
        // Complete the interrupted re-encrypt.
        staging_base = slot.base + SLOT_STAGING_OFFSET
        staging_data = nor_read(staging_base, SLOT_STAGING_SIZE)
        nor_erase_sector_range(slot.base, staging_base - 1)
        nor_write(slot.base, staging_data)
        metadata_set_re_encrypt_state(RE_STATE_IDLE)

    let image_data = storage_read_slot(slot)
    if image_data == null: return Err(ERR-STOR-READ-001)
    if image_data.len < MIN_IMAGE_SIZE: return Err(ERR-BOOT-PARSE-001)

    // ── Phase 4: Parse Header ── (all FIH-protected checks)
    state = PARSE_HEADER
    let header = parse_header(image_data)?
    // ... all header validation from §3.11.4 ...

    // ── Phase 5: Decrypt + Verify Signature ──
    state = VERIFY_SIGNATURE
    // ... decrypt to AXI SRAM, Ed25519 verify, FIH redundant verify ...
    // ... (full Gate 2 from §3.10.3) ...

    // ── Phase 6: Verify Integrity ──
    state = VERIFY_INTEGRITY
    // ... SHA-256 hash verification, FIH redundant ...

    // ── Phase 6b: Re-encrypt with KD (if first boot after OTA) ──
    // ... K_s (ECDH) → KD re-encrypt from §3.10.3 ...

    // ── Phase 7: Check Version + Anti-Rollback ──
    state = CHECK_VERSION
    // ... anti-rollback checks from §3.11.5 ...

    // ── Phase 8: Commit Version ──
    state = COMMIT_VERSION
    // ... version counter increment ...

    // ── Phase 9: Mark Slot Active ──
    state = MARK_ACTIVE
    storage_set_slot_status(slot, ACTIVE)

    // ── Phase 10: Lock Boot ──
    state = LOCK_BOOT
    boot_lock()

    // ── Phase 11: Execute ──
    state = EXECUTE
    let entry_point = payload.entry_point()
    if entry_point == null: return Err(ERR-BOOT-PARSE-007)

    // Mark that we reached EXECUTE (for boot health tracking, §3.11.3)
    metadata_set_prev_boot_phase(0x01)

    // Write boot metrics
    boot_metrics_increment()

    // Two-pass boot: IWDG cannot be software-disabled. Use RTC magic + system reset
    // so Pass 2 jumps to the application with IWDG stopped (see §3.13).
    RTC->BKP0R = IWDG_RESET_MAGIC   // 0x49574447
    NVIC_SystemReset()              // System reset → IWDG stops → Pass 2 → jump to app
```

**Slot fallback state transitions:**

```
Current State        Condition                      Next State          Action
───────────────────  ─────────────────────────────  ──────────────────  ──────────────────────────
SELECT_SLOT          Slot A selected                VERIFY_SLOT_A       Try Slot A
VERIFY_SLOT_A        All phases pass                RTC magic + reset   Two-pass → Execute Slot A app
VERIFY_SLOT_A        Any phase fails                MARK_A_INVALID      Log error, mark slot bad
MARK_A_INVALID       Slot B status in {ACTIVE,FALL,VERIFIED}  VERIFY_SLOT_B  Try Slot B
MARK_A_INVALID       Slot B status not in {ACTIVE,FALL,VERIFIED}  FAILSAFE     Both slots dead
VERIFY_SLOT_B        All phases pass                RTC magic + reset   Two-pass → Execute Slot B app
VERIFY_SLOT_B        Any phase fails                FAILSAFE            Both slots dead
```

**Prevents infinite retry loops:**

```
// If the fallback slot also fails, enter FAILSAFE (not another retry).
// This guarantees at most 2 slot attempts per boot — bounded worst case.
// The boot_attempt_count in metadata is reset to 0 before trying the
// fallback, so the fallback slot gets a full N-attempt health window
// before its own rollback.
```

**Enhanced health tracking with slot fallback:**

```
// When falling back from Slot A to Slot B:
//   1. Slot A is marked INVALID (permanent — won't be retried until re-provisioned)
//   2. boot_attempt_count is reset to 0 (fresh start for Slot B)
//   3. prev_boot_phase is cleared to 0x00
//   4. Slot B gets its own N health-report attempts before FAILSAFE

// If Slot B also fails verification:
//   1. Slot B marked INVALID
//   2. Both slots dead → FAILSAFE → Y-Modem recovery required
//   3. Y-Modem provisions one slot, which becomes ACTIVE on next boot
```

**Interaction with OTA rollback:** When the OTA library calls `ota_rollback()`, it sets `boot_target` to the fallback slot and triggers a reset. On the next boot, the fallback slot is the primary target — it's not treated as a "retry." The fallback logic in this section only activates when the primary target slot fails verification.

**Cost:** +350 bytes code (slot retry loop, fallback status checks, error logging per slot). Zero additional flash storage — uses existing metadata fields.

---

### 3.13 Decision: IWDG Hardware Watchdog with Clean Application Handoff

**Decision:** The STM32H750's Independent Watchdog (IWDG) is configured at INIT phase to enforce a hard deadline on boot completion. A software/SysTick-based watchdog is insufficient — if the CPU hangs (infinite loop, deadlock, clock failure), SysTick interrupts also stop. IWDG runs from the independent LSI RC oscillator (~32 kHz) and resets the MCU if not refreshed within the configured window.

**Critical hardware constraint:** Once started (by writing 0xCCCC to IWDG_KR), the IWDG **cannot be disabled by software**. It can only be stopped by a system reset (NVIC_SystemReset or external reset). To deliver the application a clean state with IWDG disabled, the bootloader uses a **two-pass boot**: first pass runs the full verification with IWDG protection, then triggers a system reset. The second pass detects the reset was intentional (via RTC backup register flag) and jumps directly to the application with IWDG in hardware reset state (stopped).

**Why not just leave IWDG running for the application:**
- Application may use a different watchdog strategy (WWDG, external supervisor)
- Application may have different timeout requirements (long-running tasks, sleep modes)
- IWDG consumes ~2 µA in Stop/Standby — undesirable for low-power applications
- Forcing IWDG on the application couples the bootloader's safety margin to application design

**Design — Two-pass boot with IWDG stop:**

```
// ── Pass 1: Boot with IWDG protection ──
function sbop_boot_pass1() -> Never:
    // Capture original reset cause BEFORE platform_init() clears RCC->CSR
    // HAL_Init() reads and clears RCC->CSR flags — must capture first
    // This preserves the real reset cause (POR, BOR, IWDG, NRST pin, etc.)
    // regardless of the two-pass boot — Pass 2's SFTRSTF never reaches the app
    let original_reset_cause = RCC->CSR   // Full reset flag register

    // Phase 1: Hardware Init + IWDG start
    state = INIT
    platform_init()

    // Store original reset cause in RTC backup register for application
    // BKP3R is in the backup domain — survives system reset
    // Application reads RTC->BKP3R directly — no NOR flash write needed
    RTC->BKP3R = original_reset_cause

    // Check RTC backup register: is this a post-IWDG reset?
    if RTC->BKP0R == IWDG_RESET_MAGIC:   // 0x49574447 ("IWGD")
        // We just came back from Pass 1's intentional reset
        // IWDG is now stopped (hardware reset state)

        // VBAT+POR edge case: if VBAT powers the backup domain (coin cell),
        // RTC registers survive VDD power cycles but AXI SRAM does not.
        // Verify AXI SRAM integrity before jumping — CRC-32 mismatch means
        // a POR occurred while VBAT kept RTC alive, so AXI SRAM is garbage.
        let crc_now = crc32_compute(AXI_SRAM_BASE, AXI_SRAM_INTEGRITY_LEN)  // First 4 KB
        if crc_now != RTC->BKP1R:
            // AXI SRAM corrupted — POR with VBAT-powered RTC domain
            // Clear all RTC magic and fall through to full Pass 1 verification
            RTC->BKP0R = 0
            RTC->BKP1R = 0
            // Falls through to normal Pass 1 path (IWDG init, full verify)
        else:
            // AXI SRAM intact — safe to jump directly to application
            RTC->BKP0R = 0  // Clear magic for next cold boot
            jump_to_application()  // Never returns
            // Total Pass 2: ~5 ms

    // Normal path — full boot with IWDG
    iwdg_init()
    nor_detect_and_verify()    // JEDEC ID check

    // Verify NOR Block Protect bits BEFORE any metadata read.
    // BP bits may have been left cleared by a power-loss during OTA,
    // a failed previous boot, or an attacker probing the SPI bus.
    // Metadata at 0x20_0000+ must be write-protected before we trust it.
    let sr1 = nor_spi_read_status_register()
    if (sr1 & BP2_BIT) == 0:
        // BP2 not set — metadata area is unprotected.
        // Re-apply protection immediately. This is a defensive action;
        // log the event for tamper auditing.
        tamper_log_write(TAMPER_BP_BITS_CLEARED)
        nor_spi_write_enable()
        nor_spi_write_status_register(BP2_BIT | SRP1_ENABLE)
        // Re-verify
        let sr1_v2 = nor_spi_read_status_register()
        if (sr1_v2 & BP2_BIT) == 0:
            // NOR chip not accepting BP commands — possible hardware fault or attack
            boot_enter_failsafe(ERR-HW-NOR-002)

    check_tamper_state()

    // Phases 2-10: Full verification with IWDG refresh at each step
    let result = boot_verify_slot(boot_slot)
    if result == Err:
        // Try fallback slot (§3.12)
        ...

    // Phase 11: All verification passed — stop IWDG before jumping to app
    // Reset cause is already in RTC->BKP3R (set during INIT, survives system reset).
    // Application reads RTC->BKP3R directly to determine the original reset reason.

    // Compute CRC-32 of AXI SRAM payload for Pass 2 integrity check.
    // First 4 KB covers vector table + reset handler — sufficient to detect
    // VBAT-powered POR where RTC registers survive but AXI SRAM is lost.
    let axi_crc = crc32_compute(AXI_SRAM_BASE, AXI_SRAM_INTEGRITY_LEN)  // 4 KB
    RTC->BKP1R = axi_crc

    // Method: set magic in RTC backup register, then system reset
    RTC->BKP0R = IWDG_RESET_MAGIC   // 0x49574447
    NVIC_SystemReset()              // System reset → IWDG stops → Pass 2 starts
    // Does not return
```

**IWDG configuration (800 ms timeout):**

```
// IWDG config — LSI = 32.768 kHz (RC, ±1% after calibration, ±5% over full temp range)
// Prescaler = 32 → IWDG clock = 1024 Hz → period = 0.9766 ms
// Reload value = 820 → timeout = 820 × 0.9766 ms ≈ 801 ms
// 800 ms chosen for:
//   - LSI drift over -40°C to +85°C: ±5% → worst case 778–842 ms, still > boot budget
//   - LSI drift over 0°C to +50°C:  ±1% → worst case 793–809 ms
//   - Boot worst case (first OTA boot + re-encrypt): ~190 ms → 610 ms margin
//   - Headroom for slower NOR flash, degraded SPI timing, PLL lock delay

void iwdg_init(void) {
    // Enable RTC clock for backup register access
    RCC->BDCR |= RCC_BDCR_RTCEN;

    // Unlock IWDG_PR and IWDG_RLR registers
    IWDG->KR = 0x5555;

    // Set prescaler: PR[2:0] = 0b011 → divider = 32
    IWDG->PR = 0x03;

    // Set reload value: 820 ticks (~801 ms)
    IWDG->RLR = 820;

    // Wait for prescaler + reload to be updated
    while (IWDG->SR & IWDG_SR_PVU) {}
    while (IWDG->SR & IWDG_SR_RVU) {}

    // Start IWDG (writes 0xCCCC to KR — irreversible until reset)
    IWDG->KR = 0xCCCC;

    // Immediately refresh to start with a full window
    IWDG->KR = 0xAAAA;
}

// Refresh at every phase transition and during long operations
#define IWDG_REFRESH()  (IWDG->KR = 0xAAAA)
```

**Watchdog refresh points (Pass 1 only):**

```
INIT:            Capture RCC->CSR to local var   // Before HAL_Init() clears it
                 Store in RTC->BKP3R              // Survives system reset
                 (Pass 2 only) CRC-32 verify AXI   // BKP1R vs CRC32(AXI_SRAM, 4 KB)
                 (Pass 1 only) IWDG init (start)   // Start the clock at 800 ms
SELECT_SLOT:     IWDG refresh                // After metadata read
LOAD_IMAGE:      IWDG refresh every 4 KB     // ~0.4 ms per sector read at 80 MHz
PARSE_HEADER:    IWDG refresh                // Fixed-size parse (< 1 ms)
VERIFY_SIGNATURE: IWDG refresh every 4 KB    // Streaming decrypt to AXI SRAM
  after decrypt: IWDG refresh                // After Ed25519 verify (~1.2 ms)
VERIFY_INTEGRITY: IWDG refresh               // After SHA-256 finalize
  Phase 6b re-encrypt: IWDG refresh every 4 KB NOR write
    before staging write: IWDG refresh       // Staging write ~0.4 ms per 4 KB
    before erase:        IWDG refresh        // 4 KB sector erase ~45 ms
    after copy:          IWDG refresh        // Each 4 KB page write ~0.4 ms
CHECK_VERSION:   IWDG refresh                // After SR1 reads
COMMIT_VERSION:  IWDG refresh                // After SR1 write
MARK_ACTIVE:     IWDG refresh                // After metadata write
LOCK_BOOT:       IWDG refresh                // After BP bit set
EXECUTE:         CRC-32 over AXI SRAM first 4 KB → RTC->BKP1R
                 Write RTC->BKP0R = 0x49574447
                 NVIC_SystemReset()          // IWDG stops → Pass 2 → jump to app
FAILSAFE loop:   IWDG_REFRESH()              // Each loop iteration (~1 ms)
Y-Modem:          IWDG_REFRESH()              // Before each frame + before sector erase + before page program
```

**Timeout breakdown (first boot after OTA — worst case):**

| Phase | Operation | Time | IWDG refresh? |
|---|---|---|---|
| INIT | RCC->CSR capture + BKP3R store + Clock init + IWDG start + NOR detect | ~6 ms | Start at 800 ms |
| SELECT_SLOT | Metadata read | < 1 ms | Yes (after) |
| LOAD_IMAGE | NOR read (80 B header + 16 B IV) | < 1 ms | Yes (after) |
| PARSE_HEADER | Header validation | < 1 ms | Yes (after) |
| VERIFY_SIGNATURE | AES-CTR decrypt 512 KB to AXI SRAM | ~51 ms | Every 4 KB (~0.4 ms each) |
| VERIFY_SIGNATURE | Ed25519 verify | ~1.2 ms | Yes (after decrypt, before verify) |
| VERIFY_INTEGRITY | SHA-256 (HW HASH, streaming) | ~13 ms | Yes (after finalize) |
| Phase 6b | NOR write staging area (header + IV + encrypted) | ~0.4 ms | Yes (after) |
| Phase 6b | NOR sector erase (original area) | ~45 ms | Yes (before) |
| Phase 6b | NOR copy staging → slot base (512 KB) | ~51 ms | Every 4 KB (~0.4 ms each) |
| CHECK_VERSION | SR1 reads (4-way redundant) | < 1 ms | Yes (after) |
| COMMIT_VERSION | SR1 write | ~10 ms | Yes (after) |
| MARK_ACTIVE | Metadata write | ~10 ms | Yes (after) |
| LOCK_BOOT | BP bit set via SPI | < 1 ms | Yes (after) |
| EXECUTE | Set RTC magic + NVIC_SystemReset | < 1 ms | Reset stops IWDG |
| **Total Pass 1** | | **~190 ms** | **800 ms timeout, 610 ms margin** |
| **Pass 2** | Detect RTC magic, CRC-32 check AXI SRAM, jump | **~5 ms** | IWDG stopped (reset state) |

**Normal boot (KD-encrypted, no re-encrypt needed):**

| Pass | Time | IWDG state |
|---|---|---|
| Pass 1 (full verify) | ~140 ms | Running (800 ms timeout, 660 ms margin) |
| Pass 2 (fast jump) | ~5 ms | Stopped (hardware reset state) |
| **Total to app** | **~145 ms** | IWDG disabled |

**RTC backup register usage:**

```
// RTC backup registers are in the backup domain — survive system reset,
// lost only on VBAT removal or RTC domain reset. Cold boot has all BKPx = 0.
//
// BKP0R values (two-pass boot handshake):
//   0x00000000 — Cold boot (POR/BOR). Run full Pass 1 with IWDG.
//   0x49574447 — Post-IWDG intentional reset. IWDG is stopped. Skip to Pass 2.
//
// BKP1R–BKP2R: AXI SRAM integrity CRC-32 commitment (see §3.13 VBAT+POR edge case)
//
// BKP3R: Original RCC->CSR captured before HAL_Init() clears it in Pass 1.
//        Survives system reset — application reads RTC->BKP3R directly.
//        Contains the real reset cause (POR, BOR, IWDG, NRST pin, etc.)
//        regardless of the two-pass boot process.
//   other      — Reserved. Treat as cold boot.

// This uses one of 32 RTC backup registers (4 bytes of 128 available).
// No NOR flash write needed — zero latency.
```

**Application reading the reset cause:**

```c
// In application code — read the real reset cause from RTC backup register.
// RTC->BKP3R was set by the bootloader in Pass 1 INIT before HAL_Init()
// cleared RCC->CSR. It survives the Pass 1 → Pass 2 system reset.
//
// The application must enable RTC clock before reading backup registers:
//   RCC->BDCR |= RCC_BDCR_RTCEN;
//   uint32_t reset_cause = RTC->BKP3R;

#include "stm32h7xx_hal.h"

typedef enum {
    RESET_CAUSE_UNKNOWN    = 0x00000000,  // Cold POR — BKP3R not yet written
    RESET_CAUSE_POR_BOR    = (1<<26),     // RCC_CSR_PORRSTF  — Power-on / brown-out
    RESET_CAUSE_NRST_PIN   = (1<<27),     // RCC_CSR_PINRSTF  — External NRST pin reset
    RESET_CAUSE_SOFTWARE   = (1<<28),     // RCC_CSR_SFTRSTF  — NVIC_SystemReset
    RESET_CAUSE_IWDG       = (1<<29),     // RCC_CSR_IWDGRSTF — Independent watchdog
    RESET_CAUSE_WWDG       = (1<<30),     // RCC_CSR_WWDGRSTF — Window watchdog
    RESET_CAUSE_LOW_POWER  = (1<<31),     // RCC_CSR_LPWRRSTF — Low-power reset
} reset_cause_flag_t;

static inline uint32_t app_get_reset_cause(void) {
    // RTC clock must be enabled to read backup registers
    if (!(RCC->BDCR & RCC_BDCR_RTCEN)) {
        RCC->BDCR |= RCC_BDCR_RTCEN;
    }
    return RTC->BKP3R;
}

// Usage:
//   uint32_t cause = app_get_reset_cause();
//   if (cause & RESET_CAUSE_IWDG) { ... watchdog reset ... }
//   if (cause & RESET_CAUSE_NRST_PIN) { ... external pin reset ... }
//   if (cause == RESET_CAUSE_UNKNOWN) { ... cold POR/BOR ... }
//
// Note: On the very first cold boot (factory), BKP3R = 0x00000000.
//       After bootloader Pass 1 runs, BKP3R contains the real RCC->CSR value.
//       The application should treat 0x00000000 as POR/BOR (cold boot).
```

**VBAT + POR edge case — AXI SRAM integrity protection:**

If VBAT powers the backup domain (e.g., coin cell on VBAT pin), RTC backup registers survive VDD power cycles. However, AXI SRAM is volatile and loses content on POR. This creates a dangerous edge case:

1. Bootloader Pass 1 completes verification, payload in AXI SRAM
2. Pass 1 sets `RTC->BKP0R = IWDG_RESET_MAGIC` and `BKP1R = CRC32(AXI_SRAM)`
3. NVIC_SystemReset() — IWDG stops, AXI SRAM survives (system reset only)
4. **Before Pass 2 starts**, VDD power is lost (power glitch, battery swap)
5. **If VBAT is present**: RTC registers survive → `BKP0R` still has `IWDG_RESET_MAGIC`
6. **But AXI SRAM is lost** — POR cleared it
7. On next power-up, Pass 2 would see the stale RTC magic and jump to garbage

**Protection:** Pass 2 INIT verifies `CRC32(AXI_SRAM_BASE, 4 KB) == RTC->BKP1R` before jumping. CRC-32 mismatch → treat as cold boot, clear RTC magic, fall through to full Pass 1 verification.

```
// CRC-32 coverage: first 4 KB of AXI SRAM
// Covers vector table (SP, PC, VTOR entries), reset handler, and early init.
// CRC-32 over 4 KB at 400 MHz: ~10 µs with hardware CRC, ~50 µs software.
#define AXI_SRAM_INTEGRITY_LEN  4096

// Pass 2 fast path — integrity check before jump:
if crc32_compute(AXI_SRAM_BASE, AXI_SRAM_INTEGRITY_LEN) != RTC->BKP1R:
    // AXI SRAM corrupted — POR with VBAT-powered RTC domain
    RTC->BKP0R = 0   // Clear stale magic
    RTC->BKP1R = 0
    // Fall through to full Pass 1 verification
```

**CRC-32 vs full SHA-256:** CRC-32 over 4 KB is sufficient. The payload was already cryptographically verified in Pass 1. The only question is whether AXI SRAM survived — and AXI SRAM corruption from POR is uniform (all cells discharge together). A CRC-32 mismatch definitively indicates POR occurred. SHA-256 over 512 KB would be excessive (~13 ms) for a check that almost always passes.

**Temperature drift analysis (LSI accuracy over full range):**

| Temperature | LSI Accuracy | IWDG Period | Timeout (820 ticks) | Boot Margin (190 ms) |
|---|---|---|---|---|
| +25°C (calibrated) | ±1% | 0.9766 ms | 793–809 ms | 603–619 ms |
| 0°C to +50°C | ±1.5% | 0.9766 ms | 789–814 ms | 599–624 ms |
| -40°C to +85°C | ±5% | 0.9766 ms | 778–842 ms | 588–652 ms |

Worst case boot time margin: 588 ms at -40°C (fast LSI). This is >3× the boot budget — even with degraded NOR flash performance, the margin is sufficient.

**SPI NOR presence detection (INIT phase, before SPI read):**

```
// After IWDG init, before any NOR operation:
// Read JEDEC ID (9FH command) to verify NOR flash is present and correct type
function nor_detect_and_verify():
    // Check both supported NOR flash variants:
    //   BY25Q32ES:   0x68 0x40 0x16  (Boya, 4 MB, 3× 1024 B SR)
    //   W25Q32JV:    0xEF 0x40 0x16  (Winbond, 4 MB, 3× 256 B SR)
    jedec_id = nor_spi_read_jedec_id()   // Send 0x9F, read 3 bytes
    let mfr = jedec_id[0]
    let type = jedec_id[1]
    let cap  = jedec_id[2]
    assert(mfr == 0x68 or mfr == 0xEF, ERR-HW-NOR-001)  // Boya or Winbond
    assert(type == 0x40, ERR-HW-NOR-001)                 // SPI NOR flash
    assert(cap  == 0x16, ERR-HW-NOR-001)                 // 4 MB (32 Mbit)
    // If JEDEC ID mismatch: NOR not present, wrong chip, or SPI bus fault
    // Enter FAILSAFE with ERR-HW-NOR-001
```

**Attack surface consideration:** The IWDG is configured before any external input is processed (INIT phase, before SPI or USART). A malfunctioning NOR flash that holds SPI CLK low will not block IWDG — IWDG runs from independent LSI, not from HCLK. If SPI hangs waiting for NOR response, IWDG expires → MCU reset → boot retry. The RTC magic value (0x49574447) is checked before SPI/USART init on Pass 2 — an attacker on USART1 cannot inject this value because RTC backup registers are only accessible via the APB bus.

**Application start state:** After Pass 2, the application begins execution with IWDG in hardware reset state (disabled, KR=0x0000, PR=0x00, RLR=0x0FFF). The application is free to:
- Leave IWDG disabled (no watchdog overhead, no Stop/Standby current drain)
- Configure its own IWDG with application-appropriate timeout
- Use WWDG instead (window watchdog, PCLK-based, software-disableable)
- Use an external hardware watchdog supervisor

**Cost:** +200 bytes code (IWDG init, refresh macros, NOR detect, RTC magic, Pass 2 fast path). Zero flash storage. Uses 1 of 32 RTC backup registers. Uses existing LSI oscillator (always running on STM32H750).

---

### 3.14 OTA Download Progress Journal

**Problem:** OTA downloads over unreliable networks (cellular, LoRa, BLE) can be interrupted at any time. Without checkpointing, every interruption requires restarting the download from byte 0. For a 512 KB image over a 50 Kbps link (~82 seconds), a failure at 90% progress wastes ~74 seconds of download time and ~370 KB of bandwidth. On metered or power-constrained devices, this is unacceptable.

MCUboot does not implement download resume natively — it relies on the application layer (MCUmgr/SMP) to handle interruption recovery. SBOP can improve on this by providing a built-in progress journal that survives power loss and enables HTTP Range resume without application involvement.

**Design:** A 4 KB progress journal sector at NOR 0x20_3000 stores checkpoints as 64-byte records in a ring buffer. Each checkpoint records the total bytes written and a truncated running hash (first 8 bytes of SHA-256). On resume, the bootloader verifies existing data integrity against the stored hash before issuing an HTTP Range request from the last checkpoint offset.

```
Progress record (64 bytes):
┌────────────────────────────┐
│ [0]  magic          : u32  │ = 0x4F544150 ("OTAP")
│ [4]  sequence       : u32  │ Chunk counter (0, 1, 2, ...)
│ [8]  target_slot    : u8   │ SLOT_A or SLOT_B
│ [9]  flags          : u8   │ Reserved
│ [10] reserved       : u8[2]│
│ [12] bytes_written  : u32  │ Total bytes written so far
│ [16] total_size     : u32  │ Expected image size (from backend)
│ [20] expected_hash  : u8[32]│ SHA-256 of full image
│ [52] running_hash   : u8[8] │ Truncated SHA-256 of [0..bytes_written)
│ [60] crc32          : u32  │ CRC-32 over bytes 0..59
└────────────────────────────┘
```

**Checkpoint interval:** Every 32 KB of downloaded data. For a 512 KB image: 16 checkpoints per download, 4 full downloads per sector erase (64 records/sector × 1 record / 32 KB = 2,048 KB/sector).

**Resume sequence:**

```
// On download start (or retry after interruption):
function ota_download_with_resume(slot, info):
    let prev = ota_progress_read_last()
    let offset = 0

    if prev.valid and prev.target_slot == slot
       and prev.expected_hash == info.image_hash:
        // Verify existing data before trusting checkpoint
        let existing = storage_read_slot_range(slot, 0, prev.bytes_written)
        if sha256(existing)[0..8] == prev.running_hash:
            offset = prev.bytes_written
            log("OTA resume: {} B already written", offset)
        else:
            // Data corrupt — restart
            storage_erase_slot(slot)
            ota_progress_clear()

    // HTTP Range request from offset
    let handle = http_get(info.url, headers={"Range": "bytes={}-".format(offset)})

    // Download remaining data with periodic checkpoints
    // ... (download loop with ota_progress_write every 32 KB) ...

    // Success — clear progress journal
    ota_progress_clear()
```

**Checkpoint write protocol (append-only, no erase until wrap):**

```
function ota_progress_write(bytes_written, running_hash):
    let write_off = ota_progress_find_next_free()
    if write_off >= 4096:
        nor_spi_erase_sector(0x20_3000)  // Ring wrapped
        write_off = 0
    // Build and write 64 B record with CRC-32
    nor_spi_page_program(0x20_3000 + write_off, record, 64)
```

**Read protocol (scan backward for last valid record):**

```
function ota_progress_read_last() -> Option<ProgressRecord>:
    for off in (4096 - 64) down to 0 step 64:
        data = nor_spi_read(0x20_3000 + off, 64)
        if data[0..4] == 0x4F544150:           // Magic
            if crc32(data[0..60]) == data[60..64]:
                return parse(data)
    return None  // Sector empty or all records corrupt
```

**Security considerations:**

| Concern | Mitigation |
|---------|-----------|
| Attacker writes fake progress record to redirect download | Progress record doesn't control execution — only saves bytes_written. Resume requires matching expected_hash and running_hash. Fake record with wrong hash → restart from 0. |
| Progress journal replay (old record from previous update) | expected_hash must match current UpdateInfo.image_hash. Every OTA has a unique hash → old records are silently ignored. |
| Truncated running hash (8 bytes = 64 bits) | Sufficient for integrity check, not authentication. An attacker constructing data with the same truncated hash needs 2^64 attempts. Even if they succeed, the full SHA-256 verification at the end of download catches any discrepancy. |
| Power loss during checkpoint write | CRC-32 catches partial writes. read_last() skips corrupt record and finds previous valid one. |

**Why 8-byte running hash instead of full SHA-256:** The running hash is a data-integrity check, not a security boundary. Its purpose is to detect NOR flash corruption (stuck bits, partial writes) — not to authenticate the image. The full SHA-256 verification at the end of Phase 3 provides cryptographic authentication. An 8-byte truncated hash makes the record compact (64 bytes, clean alignment) while providing sufficient integrity detection: probability of undetected corruption < 2^-64 ≈ 5.4 × 10^-20.

**Cost:** One 4 KB NOR sector at 0x20_3000, +~350 bytes code. Effectively free at boot time (not on the boot path — only used during OTA).

**MCUboot comparison:** MCUboot delegates download resume to the application layer (MCUmgr/SMP). This is flexible but requires every application to implement its own checkpointing. SBOP's built-in progress journal provides resume capability at the platform level — any OTA client (HTTPS, CoAP, BLE, LoRa) benefits without application-level changes.

---

### 3.15 Measured Boot with Chained PCR Measurements

**Problem:** Without measured boot, there is no way to prove to a remote verifier what firmware actually booted on the device. A compromised device can claim to be running any version. Remote attestation requires a tamper-evident log of boot events, where each event extends a cryptographic chain — modifying any event breaks the chain, making forgery computationally infeasible.

MCUboot supports measured boot via BOOT_RECORD TLVs stored in the image TLV area, integrated with PSA Crypto's `psa_measure_boot()` API. SBOP adopts the same concept but stores measurements in the frequent journal (alongside other per-boot records) rather than in the image slot, enabling the measurement log to be read without parsing image TLVs.

**Design:** Each boot extends a SHA-256 PCR (Platform Configuration Register) chain. The initial PCR is computed from the bootloader version and device identity. Each subsequent boot extends the chain with the booted image hash, slot, and boot result. The PCR value is stored in the FrequentRecord at offset 32.

```
PCR chain:

PCR[0] = SHA-256("SBOP_MEASURED_BOOT_V1" || bootloader_version || device_uid)
PCR[1] = SHA-256(PCR[0] || image_hash[0] || SLOT_A || PHASE_EXECUTE)
PCR[2] = SHA-256(PCR[1] || image_hash[1] || SLOT_A || PHASE_EXECUTE)
PCR[3] = SHA-256(PCR[2] || image_hash[2] || SLOT_B || PHASE_CONFIRMED)
...

Where:
  bootloader_version : u32   = SBOP bootloader build version
  device_uid         : u8[16]= DeviceID.uid (unique per device)
  image_hash[N]      : u8[32]= SHA-256 of firmware image for boot N
  slot               : u8    = SlotID (SLOT_A=0x00, SLOT_B=0x01)
  boot_phase         : u8    = 0x01 (EXECUTE) or 0x02 (CONFIRMED)
```

**Boot flow integration (Phase 11 EXECUTE):**

```
let prev = frequent_journal_read_last()
let prev_pcr = prev.valid ? prev.pcr_value : measured_boot_initial_pcr()
let new_pcr = sha256(prev_pcr || header.hash || slot as u8 || 0x01)
frequent_journal_append(0x01, boot_flags, iv_hash, new_pcr)
```

**Initial PCR computation:**

```
function measured_boot_initial_pcr() -> [u8; 32]:
    return sha256("SBOP_MEASURED_BOOT_V1"
                  || BOOTLOADER_VERSION as u32 LE
                  || device_uid[0..16])
```

**Remote attestation verification:**

A remote verifier (backend attestation service) validates the device's boot history by recomputing the PCR chain from the initial value:

```
// Verifier knows:
//   - bootloader_version (from device registration)
//   - device_uid (from attestation request)
//   - expected image hashes for each authorized firmware version

let computed_pcr = sha256("SBOP_MEASURED_BOOT_V1"
                          || device.bootloader_version
                          || device.device_uid)

for each record in device.measurement_log:
    computed_pcr = sha256(computed_pcr
                          || record.image_hash
                          || record.slot
                          || record.boot_phase)

if computed_pcr == device.current_pcr:
    attestation_result = PASS
else:
    attestation_result = FAIL  // Measurement log tampered or unexpected boot
```

**Tamper-evidence property:** The SHA-256 chain makes it computationally infeasible to modify or remove a measurement record without detection. Changing record[N] changes PCR[N], which changes PCR[N+1], PCR[N+2], ..., all the way to the current PCR. The only way to forge a valid chain is to find a SHA-256 preimage (2^256 work factor) or recompute all subsequent hashes with the modified data (which requires knowing the correct image hashes for every boot).

**Measurement log retrieval:** The application reads the frequent journal via a system call (`boot_get_measurement_log()`) and sends the records to the backend attestation service. The backend verifies the PCR chain and confirms the device booted only authorized firmware.

```
// Application attestation client:
let measurements = boot_get_measurement_log()  // Returns FrequentRecord[]
let attestation_report = {
    device_uid: identity_get_device_id(),
    current_pcr: measurements.last().pcr_value,
    measurement_count: measurements.len(),
    measurement_log: measurements,
    signature: ed25519_sign(KD_Auth, attestation_report),
}
http_post(attestation_url, attestation_report)
```

**Storage and performance:**

| Parameter | Value |
|-----------|-------|
| Record size | 64 bytes (was 32 B before measured boot) |
| Records per sector | 64 (was 128) |
| Sector erase interval | ~64 boots (at one boot/day: ~2 months) |
| PCR computation cost | SHA-256 over 32+32+1+1 = 66 bytes per boot |
| Attestation verification cost | SHA-256 over 66 bytes per boot in log (parallelizable) |

**Comparison with MCUboot measured boot:**

| Feature | MCUboot | SBOP |
|---------|---------|------|
| Measurement storage | BOOT_RECORD TLV in image slot | Frequent journal (separate metadata area) |
| PCR algorithm | SHA-256 (PSA Crypto API) | SHA-256 (chained, same algorithm) |
| Measurement types | SW_COMPONENT, SW_COMPONENT_CONFIG, etc. | Single unified measurement per boot |
| API | `psa_measure_boot()` | `boot_get_measurement_log()` (application SVC) |
| TCG/TPM alignment | Partial (PSA-style, not TPM PCR) | Same approach — PSA-style chained measurements |
| Verification | Application-read + PSA attestation token | Application-read + Ed25519-signed attestation report |

SBOP's measured boot is simpler than MCUboot's — a single measurement per boot rather than separate measurements for each software component and configuration. This is appropriate for a first-stage bootloader where the only measured components are the bootloader itself and the application image. Multi-stage bootloaders would extend the chain in each stage.

**Cost:** The FrequentRecord grows from 32 to 64 bytes (+32 bytes). This halves the ring buffer capacity (128→64 records) but still provides ~2 months of daily-boot history. Code increase is negligible — the SHA-256 extend operation reuses the existing crypto engine. Zero additional NOR sectors.

## 4. Flash Layout

### 4.1 Internal Flash (128 KB) — Zone 1 Bootloader

```
Address         Size    Region                  Contents
─────────────── ─────── ─────────────────────── ──────────────────────────────────
0x0800_0000     512 B   Vector Table            Initial SP, Reset_Handler, NMI,
                                                 HardFault (FIH-safe), MemManage
0x0800_0200     4 KB    .fih_constants           FIH_SUCCESS, FIH_FAILURE, FIH_MASK
0x0800_1200     37 KB   .text (bootloader)       State machine, SPI NOR driver, image parser,
                                                 Ed25519 verify, SHA-256, AES-CTR decrypt/encrypt,
                                                 AXI SRAM copy, Y-Modem Rx (USART1), FIH checks,
                                                 error handler, ECDH→KD re-encrypt logic
0x0800_A600     8 KB    .text (PC-ROP secured)   KD access / ECDH (X25519) / KD_Auth derive /
                                                 (SVC handler, commitment verify)
                                                 (execute-only, no read from outside)
0x0800_C600     8 KB    .rodata                  Error strings, key references, device config
                                                 (Multi-key CSM mode: NO public keys in .rodata —
                                                 keys embedded in image SignatureBlock, verified
                                                 against SR1 OTP fingerprints at boot)
0x0800_E600     4 KB    .data                    Initialized data — boot state
0x0800_F600     8 KB    .bss + stack             Runtime state, 4 KB stack
0x0801_1600     2 KB    Option Bytes             ROP, PC-ROP, BFB2, Secure Access cfg
0x0801_1E00     ~57 KB  Reserved                 Future expansion / second-stage
─────────────── ─────── ─────────────────────── ──────────────────────────────────
Total: 128 KB
```

**PC-ROP configuration:** Sectors at 0x0800_A600–0x0800_E5FF are marked execute-only. Any read from non-PCROP code (including Zone-2 application) returns zero. The KD-related crypto code, SVC handler, and the embedded public keys reside here.

**I-Cache:** Enabled at INIT phase. The 16 KB I-Cache covers the bootloader hot path (verification loop, SPI read loop) with ~95% hit rate. Internal flash latency drops from 5-7 wait states to effectively zero for cached instruction fetch.

**D-Cache:** Disabled. Bootloader data is in DTCM (128 KB, zero-wait, deterministic) — stack, .data, .bss, FIH state, and SPI buffers all reside here. No repeated data access pattern exists that would benefit from D-Cache. Keeping D-Cache off avoids coherency concerns: when the bootloader writes the application image to AXI SRAM and then jumps to it, there is no stale cache to flush. The AXI SRAM copy is a streaming write — D-Cache would only add flush overhead with zero benefit.

### 4.2 External NOR Flash (4 MB) — Zone 2 Firmware + Metadata

```
Byte Address    Size      Region                  Contents
─────────────── ───────── ─────────────────────── ──────────────────────────────────
0x00_0000       1 MB      Slot A                  Firmware image (at-rest encrypted with KD):
                                                    ImageHeader (80 B, plaintext)
                                                    IV_dev (16 B, plaintext)
                                                    AES-256-CTR(KD, IV_dev, Payload) ≤ 512 KB
                                                    InnerSignatureBlock (116 B, plaintext)
                                                    Padding (~512 KB headroom)
0x10_0000       1 MB      Slot B                  Firmware image (identical layout)
0x20_0000       128 KB    Slot Metadata Area      Critical Journal Copy A (4 KB sector, 128 B used)
                                                   Critical Journal Copy B (4 KB sector at 0x20_1000)
                                                   Frequent Journal ring buffer (4 KB sector at 0x20_2000)
                                                   OTA Progress Journal (4 KB sector at 0x20_3000)
                                                   Error Log (4 KB circular buffer at 0x20_4000)
                                                   Reserved
0x22_0000       4 KB      Board Information Area  board_id (u16), board_name (char[32]), board_serial (char[32]), system_name (char[32]), system_serial (char[32]), config_flags (u32), crc32, reserved. Factory-provisioned, static.
0x22_1000       4 KB      KV Storage Area         Bootloader region (2 KB): board info populated at boot.
                                                   Application region (2 KB): user/application config KV pairs.
                                                   TLV format with CRC-16 per entry.
0x22_2000       ~1.867 MB Reserved                Future: second-stage loader,
                                                   encrypted delta patches,
                                                   debug auth challenge log
0x3F_F000       4 KB      Security Register Area  (accessible only via 48H/42H/44H)
                                                   SR1: version counter (4× 32-bit) + signing key fingerprints (4× 32 B) + sig key revocation. 256 B max.
                                                   SR2: OTA key fingerprints (4× 32 B) + OTA key revocation + KD commitment (32 B). 256 B max.
                                                   SR3: NOR-MCU binding (224-bit composite UID hash, 256 B max)
                                                   Cross-vendor: only first 256 B per SR used
─────────────── ───────── ─────────────────────── ──────────────────────────────────
Total: 4 MB
```

**Slot content layout (1 MB each, after first boot following OTA):**

```
// Slot sizing constants
SLOT_SIZE            = 1 MB    (0x10_0000)
SLOT_STAGING_OFFSET  = 512 KB  (0x08_0000)  // Upper half of slot for write-before-erase staging
SLOT_STAGING_SIZE    = 512 KB               // Maximum staging data (header + IV + payload + sig)
```

```
Offset      Size    Field                   Description
─────────── ─────── ─────────────────────── ──────────────────────────────────
0x000000    80 B    ImageHeader             Magic, version, size, board_id, flags, hash, enc_key_index, header_hmac
                                           FLAG_OTA_PENDING = 0 (re-encrypted with KD)
0x000050    16 B    IV_dev                  Random IV for KD encryption
0x000060    ≤512 KB EncryptedPayload        AES-256-CTR(KD, IV_dev, Payload)
                                           KD = permanent device key
                                           Payload = plaintext application binary
0x080060    116 B   InnerSignatureBlock     Ed25519(KI_private) over (ImageHeader||Payload)
                                           Outside encryption envelope (plaintext)
0x0800D4    ~512 KB Padding / Reserved      Headroom for larger second-stage images
                                           **Upper half (≥0x8_0000) reserved as write-before-erase staging area**
─────────── ─────── ─────────────────────── ──────────────────────────────────
Total: 1 MB per slot

During OTA download (before first boot):
  Payload encrypted with K_s via X25519 ECDH
  ImageHeader.flags has FLAG_OTA_PENDING set

After first boot (bootloader re-encrypts):
  Payload encrypted with KD (permanent)
  ImageHeader.flags has FLAG_OTA_PENDING cleared
  K_s zeroized — never needed again
```

**Slot metadata layout (tiered journal, NOR 0x20_0000):**

Metadata is split into two independent journals with different update frequencies. The critical journal (slot state, version ceiling) uses two-copy atomic writes in separate 4 KB sectors. The frequent journal (per-boot counters, IV hash) uses a simple ring buffer in its own sector. See §3.11.1 for detailed write/read protocol.

```
Critical Journal — Copy A (0x20_0000, 4 KB sector):
Offset   Size  Field
───────  ────  ─────────────────────────────────
0x000    4 B   sequence : u32
               (odd = writing, even = committed)
0x004    1 B   active_slot
0x005    1 B   boot_target
0x006    1 B   re_encrypt_state
               (0x00 = idle, 0x01 = staged, 0x02 = committed)
0x007    1 B   reserved
0x008   52 B   slot_a_info (ImageInfo: status, version, hash, flags, timestamp, boot_count)
0x03C   52 B   slot_b_info (ImageInfo)
0x070    4 B   max_allowed_version
0x074    8 B   reserved2
0x07C    4 B   crc32 (over bytes 0x000–0x07B)
───────  ────
128 bytes total. Rest of 4 KB sector unused.

Critical Journal — Copy B (0x20_1000, separate 4 KB sector):
Offset   Size  Field
───────  ────  ─────────────────────────────────
0x000    4 B   sequence : u32
0x004  124 B   (identical fields to Copy A bytes 0x004–0x07C)
───────  ────
128 bytes total. Rest of 4 KB sector unused.

Frequent Journal — Ring Buffer (0x20_2000, 4 KB sector):
Offset   Size  Field
───────  ────  ─────────────────────────────────
0x000   64 B   Record 0: {boot_seq(u32), prev_boot_phase(u8), flags(u8),
               reserved(u8[2]), last_iv_hash(u8[24]), pcr_value(u8[32])}
0x040   64 B   Record 1
  ...    ...   ...
0xFC0   64 B   Record 63
───────  ────
64 records × 64 bytes = 4 KB. Append-only ring buffer.
CRC-16-CCITT per record at bytes 62-63. Scan backward for last valid record.
Erased once per ~64 boots when ring wraps.
pcr_value provides chained measurement for remote attestation (measured boot).

Remaining metadata region:
0x20_3000   4 KB   ota_progress             OTA download progress journal (ring buffer, 64 B × 64 records)
                                            Magic: 0x4F544150 ("OTAP"). Records: sequence, target_slot,
                                            bytes_written, total_size, expected_hash, running_hash, CRC-32.
                                            Erased on download completion. Survives power loss for resume.
0x20_4000   4 KB   error_log                Circular error log buffer
0x20_5000  ~108 KB reserved                 Future expansion
```

**Board Information Area (4 KB sector at NOR 0x22_0000):**

Programmed once at factory provisioning (not OTP — config fields may be updated in the field via authenticated service command). Protected by BP bits alongside metadata after boot.

```
Offset      Size    Field                   Description
─────────── ─────── ─────────────────────── ──────────────────────────────────
0x0000      4 B     magic_board_info        Magic: 0x424F4944 ("BOID")
0x0004      2 B     board_id                SKU identifier (e.g., 0x0001 = H750VBT-SBOP-R1)
0x0006      2 B     board_revision          Hardware revision (e.g., 0x0100 = Rev 1.0)
0x0008      32 B    board_name              Null-terminated ASCII (e.g., "STM32H750VBT-SBOP-REV-A")
0x0028      32 B    board_serial            Unique per-board serial (e.g., "SN-20260428-0001")
0x0048      32 B    system_name             Product name (e.g., "SBOP-Controller-V1")
0x0068      32 B    system_serial           Unique per-system serial (may differ from board_serial
                                            if the system contains multiple boards)
0x0088      4 B     config_flags            Configuration flags:
                                              bit 0: debug_enabled (1 = debug port allowed after auth)
                                              bit 1: factory_test_mode (1 = EOL test in progress)
                                              bit 2: recovery_mode (1 = Y-Modem always available)
                                              bit 3: boot_verbose (1 = USART1 boot log output)
                                              bits 4-31: reserved (must be 0)
0x008C      128 B   config_data             Variable configuration area (JSON-compatible key-value
                                            pairs, null-terminated, e.g.:
                                            "backend_url=https://ota.example.com\0"
                                            "update_interval=3600\0"
                                            "\0" = end of config)
0x010C      116 B   reserved2               Must be zero
0x0180      4 B     crc32_board_info        CRC-32 over bytes 0x0000–0x017F
0x0184      ~3.6 KB padding                 Reserved (4 KB sector total)
─────────── ─────── ─────────────────────── ──────────────────────────────────
Total: 4 KB (one 4 KB erase sector)
```

**Board ID verification flow:**

```
Bootloader Phase 4 (PARSE_HEADER):
  1. Read board_id from NOR Board Information Area (0x22_0004, 2 bytes)
  2. Redundant read (FIH): read again, compare both reads match
  3. Compare header.board_id against stored board_id
  4. If mismatch → ERR-BOOT-PARSE-008 → FAILSAFE
  5. If match → proceed to Phase 5

This prevents:
  - Loading firmware built for a different product SKU
  - Loading firmware built for a different board revision (if revision-gated)
  - Loading generic unsigned firmware (board_id = 0 is invalid)

Backend enforces at build time:
  - Signing tool reads board_id from board manifest
  - board_id is embedded in ImageHeader, then signed (part of sig_input)
  - Backend won't serve an image with wrong board_id to this device
```

**Board Information Area write protection:**

```
After boot (Phase 10, LOCK_BOOT):
  // BP bits already protect upper 2 MB (metadata 0x20_0000 + board info 0x22_0000)
  // Board Info Area is hardware write-protected during normal operation

Application runtime (if config update needed):
  // Must authenticate via KD_Debug challenge-response
  // Then temporarily unlock BP, update config, re-lock
  svc_board_config_update(auth_token, key, value);
  // SVC handler verifies auth, unlocks BP, writes, re-locks
```

**Boot decrypt + copy flow (Phase 5–11):**

Decryption happens in a single streaming pass during Phase 5 (VERIFY_SIGNATURE). The plaintext Payload goes directly to AXI SRAM (volatile). It never touches NOR flash. Power loss at any point leaves only ciphertext in NOR.

```
Phase 5 (VERIFY_SIGNATURE) + Phase 6 (VERIFY_INTEGRITY) — combined streaming pass:

  Input:  NOR slot → ImageHeader (80 B) + E (32 B) + IV (16 B) + AES-CTR(Payload) + InnerSigBlock
  Output: AXI SRAM → Payload (plaintext, ready to execute)

  // For first boot after OTA (FLAG_OTA_PENDING set):
  1a. enc_key_index = header.reserved[0]
  1b. pub = X25519(OTA_private[enc_key_index], G); verify SHA-256(pub) against SR2 OTP
  1c. Check SR2 OTA revocation bitmap for enc_key_index
  1d. E = nor_read(slot_base + 80, 32)                             // Ephemeral public key
  1e. S = X25519(OTA_private[enc_key_index], E)                     // ECDH shared secret
  1f. K_s = HKDF-SHA-256(S, "SBOP-OTA-ENC", 32)                   // AES-256 session key
  1g. secure_zeroize(S, 32)

  // For subsequent boots (FLAG_OTA_PENDING clear):
  1.  Skip ECDH — use KD directly. IV is at slot_base + 80.

  // IV reuse detection (prevents AES-CTR keystream replay):
  // If same (KD, IV_dev) pair is used twice, XOR of two ciphertexts = XOR of plaintexts.
  // Detect by storing hash of last-used IV.
  1h. iv_current = nor_read(slot_base + 80, 16)
  1i. iv_hash = sha256(iv_current || kd_identifier)       // KD_identifier: first 16 B of KD
  1j. last_record = frequent_journal_read_last()
  1k. if last_record.valid:
          assert(constant_time_compare(iv_hash[0..24], last_record.last_iv_hash, 24) == false,
                 ERR-BOOT-CRYPTO-007)                     // IV reuse detected - possible replay
  1l. // IV hash recorded in Phase 11 via frequent_journal_append()
      // (ring buffer, 32 B record - no sector erase needed)
  // Common decrypt path:
  2.  Configure HW CRYP: AES-256-CTR, key = K_s or KD, IV from NOR
  3.  Configure HW HASH: SHA-256, start
  4.  Streaming loop (CHUNK_SIZE = 4096 B, DTCM buffer):
      for offset in 0..encrypted_payload_size step CHUNK_SIZE:
          ciphertext_chunk = nor_read(ciphertext_offset + offset, CHUNK_SIZE)
          plaintext_chunk  = aes_ctr_decrypt(crypto, ciphertext_chunk)
          hash_update(hash, plaintext_chunk)
          axi_sram_write(AXI_SRAM_BASE + offset, plaintext_chunk)
  5.  final_hash = hash_finalize(hash)
  6.  Parse decrypted InnerSignatureBlock from DTCM → key_index, public_key, signature
  7.  // Signing key fingerprint + revocation (NXP CSM-style multi-key)
      let expected_fp = nor_spi_read_sr1(SIG_KEY_FINGERPRINT_BASE + key_index * 32, 32)
      assert(constant_time_compare(sha256(public_key), expected_fp, 32), ERR-BOOT-CRYPTO-004)
      let revocation = nor_spi_read_sr1_u16(SIG_KEY_REVOCATION_OFFSET)
      assert((revocation >> key_index) & 1 == 1, ERR-BOOT-CRYPTO-004)
  8.  Ed25519_verify(ImageHeader || Payload, signature, public_key)
  9.  FIH: key fingerprint check + Ed25519_verify again, compare results
  10. assert(final_hash == header.hash)
  11. secure_zeroize(K_s)  // if first boot — CRYP key cleared
  12. secure_zeroize(DTCM decrypt buffer) // Chunk buffer cleared

Phase 11 (EXECUTE) — plaintext already in AXI SRAM:
  1. secure_zeroize(DTCM decrypt chunk buffer)  // Chunk buffer + InnerSigBlock parsing area
  2. RTC->BKP3R = original_reset_cause          // Reset cause for application (captured in INIT)
  3. RTC->BKP1R = crc32(AXI_SRAM_BASE, 4096)   // AXI SRAM integrity commitment
  4. RTC->BKP0R = IWDG_RESET_MAGIC             // Two-pass handshake
  5. NVIC_SystemReset()                         // IWDG stops → Pass 2 → jump_to_prepared_application()
  // Pass 2 handles VTOR, MSP, PC setup and workspace zeroization
```

**Why this is safe against power-off attack:**

```
Timeline:
  OTA complete → NOR has: ImageHeader(plain) + IV + AES-CTR(Payload) + InnerSigBlock
  Power off   → NOR: ciphertext survives
               → AXI SRAM: empty (volatile, lost power)
               → DTCM: empty (volatile, lost power)
  Power on    → Bootloader decrypts to AXI SRAM (300 ms window)
               → Plaintext exists only in AXI SRAM while device is powered
               → Attacker must be actively probing AXI SRAM during boot
               → Much harder than reading NOR flash at leisure

  Old design: plaintext in NOR for minutes/hours → trivial chip-off extraction
  New design: plaintext in AXI SRAM for ~300 ms during boot → requires
              active probing of DDR bus at exact boot moment
```

---

### 4.3 KV Storage Area (NOR 0x22_1000, 4 KB)

The KV Storage Area is a runtime key-value store shared between the bootloader (Zone 1) and the application (Zone 2). The bootloader populates board information and boot data into the BL region before jumping to the application. The application reads this data and can persist its own configuration in the APP region.

**Design rationale:** The Board Information Area (§4.2) is factory-provisioned and mostly static — it holds manufacturing-time data (board serial, SKU). The KV Storage Area serves a different purpose: it is the runtime handoff mechanism from bootloader to application, plus application persistent config. Separating them avoids mixing factory data (which should rarely change) with runtime data (which changes every boot).

#### 4.3.1 Area Layout

```
KV Storage Area (4 KB sector, NOR 0x22_1000):
─────────────────────────────────────────────────────────────
0x0000    8 B     Area Header            magic (u32), version (u16), bl_count (u8), app_count (u8)
0x0008    8 B     reserved               must be zero
─────────────────────────────────────────────────────────────
0x0010    2032 B  Bootloader Region      BL-populated entries (board info, boot data)
                  (read-only to app)     Each entry: [key:u16][flags:u8][len:u16][value][crc16:u16]
─────────────────────────────────────────────────────────────
0x0800    2048 B  Application Region     Application config entries (user settings, runtime state)
                  (read-write to app)    Each entry: [key:u16][flags:u8][len:u16][value][crc16:u16]
─────────────────────────────────────────────────────────────
Total: 4 KB (one 4 KB erase sector)
```

**Area Header (8 bytes):**

```
Offset   Size  Field        Description
───────  ────  ───────────  ──────────────────────────────────
0x00     4 B   magic        Magic: 0x4B563031 ("KV01")
0x04     2 B   version      Format version (1 = initial)
0x06     1 B   bl_count     Number of valid entries in BL region
0x07     1 B   app_count    Number of valid entries in APP region
0x08     8 B   reserved     Must be zero
```

#### 4.3.2 KV Entry Format

Each entry is a self-describing TLV (Tag-Length-Value) with integrity check:

```
Offset   Size  Field        Description
───────  ────  ───────────  ──────────────────────────────────
0x00     2 B   key          Key identifier (u16, little-endian)
0x02     1 B   flags        bit 0: valid (1 = entry in use)
                            bit 1: bootloader_owned (1 = written by BL, app read-only)
                            bit 2: deleted (1 = tombstone, skip on read)
                            bits 3-7: reserved (must be 0)
0x03     2 B   value_len    Value length in bytes (u16, little-endian, 0 = empty/deleted)
0x05     N B   value        Value payload (N = value_len)
0x05+N   2 B   crc16        CRC-16-CCITT over bytes 0x00..(0x04+N)
                            (key || flags || value_len || value)
───────────────────────────────────────────────────────────────
Entry size: 7 + value_len bytes
```

**Entry operations:**
- **Add:** Append new entry at first free space (flags & 0x01 == 0). Update region count.
- **Update:** Mark old entry as deleted (flags |= 0x04), append new entry. If insufficient space, compact region first.
- **Delete:** Set flags bit 2 (deleted tombstone) and value_len = 0.
- **Read:** Scan region linearly, skip deleted entries. Match by key ID. Return first valid match (last-write-wins since new entries are appended).
- **Compact:** When region is full, erase sector, rewrite only valid entries, update header. BL region preserved across compaction.

#### 4.3.3 Well-Known Keys

Keys 0x0001–0x00FF are reserved for bootloader/system use. Keys 0x0100–0xFFFF are available for application use.

**Bootloader-populated keys (BL region, flags = 0x03 = valid | bootloader_owned):**

| Key ID | Name | Type | Description |
|--------|------|------|-------------|
| 0x0001 | BOARD_ID | u16 | SKU identifier (copied from Board Info Area) |
| 0x0002 | BOARD_REVISION | u16 | Hardware revision (copied from Board Info Area) |
| 0x0003 | BOARD_NAME | string | Board name (copied, max 32 bytes, null-terminated) |
| 0x0004 | BOARD_SERIAL | string | Board serial number (copied, max 32 bytes, null-terminated) |
| 0x0005 | SYSTEM_NAME | string | System/product name (copied, max 32 bytes, null-terminated) |
| 0x0006 | SYSTEM_SERIAL | string | System serial number (copied, max 32 bytes, null-terminated) |
| 0x0010 | BOOT_SLOT | u8 | Currently booted slot (0 = SLOT_A, 1 = SLOT_B) |
| 0x0011 | BOOT_VERSION | u32 | Bootloader version (BCD: 0x00010000 = v1.0.0) |
| 0x0012 | ACTIVE_FW_VERSION | u32 | Active firmware version (from ImageHeader) |
| 0x0013 | LAST_BOOT_REASON | u8 | 0 = normal, 1 = OTA, 2 = rollback, 3 = recovery, 4 = failsafe |
| 0x0014 | BOOT_TIMESTAMP | u32 | RTC timestamp at boot (seconds since epoch, 0 if no RTC) |
| 0x0015 | BOOT_COUNT | u32 | Total boot count (monotonic, incremented each boot) |
| 0x0016 | ROLLBACK_COUNT | u32 | Total rollback events |
| 0x0017 | CONFIG_FLAGS | u32 | Configuration flags (copied from Board Info Area, may be runtime-modified) |
| 0x0018 | FW_HASH_ACTIVE | u8[32] | SHA-256 of currently active firmware payload |
| 0x0019 | NOR_FLASH_SIZE | u32 | Detected NOR flash size in bytes |
| 0x001A | MCU_UID | u8[12] | MCU 96-bit unique ID |

**Application keys (APP region, flags = 0x01 = valid, bootloader_owned = 0):**

Keys 0x0100+ are free for application use. Suggested conventions:

| Key Range | Purpose |
|-----------|---------|
| 0x0100–0x01FF | Network configuration (backend URL, TLS settings) |
| 0x0200–0x02FF | OTA configuration (update interval, channel, policy) |
| 0x0300–0x03FF | Application settings (feature flags, calibration data) |
| 0x0400–0x04FF | User configuration (preferences, display, locale) |
| 0x0500–0xFFFF | Free |

#### 4.3.4 Bootloader → Application Handoff Flow

```
Phase 11 (EXECUTE) — after image verification, before jump:

  1. Erase KV Storage Area sector (NOR 0x22_1000, 4 KB)
     // Fresh KV area each boot — entries are deterministic from board info + boot state

  2. Write Area Header:
     magic = 0x4B563031, version = 1, bl_count = 0, app_count = 0

  3. Populate BL region with board info entries:
     for each well-known key 0x0001–0x001A:
         read source data (Board Info Area, boot state, ImageHeader)
         kv_write(key, flags=0x03, value)
         bl_count++

  4. Update Area Header with final bl_count

  5. (Optional) Check if APP region has persisted entries from previous boot:
     // APP region is preserved across KV area erase — see §4.3.5
     // If app region persistence is enabled, re-write APP entries after erase

  6. Continue to application execution
```

**Application read API (libsbop-kv):**

```c
// Read a KV entry. Returns value_len on success, 0 if not found, negative on error.
// For BL-owned keys: reads directly from NOR KV area (mapped read-only).
// For APP-owned keys: reads from NOR KV area.
int kv_read(uint16_t key, uint8_t *value, uint16_t max_len);

// Read a typed value with size verification.
static inline int kv_read_u16(uint16_t key, uint16_t *out) {
    return kv_read(key, (uint8_t *)out, 2) == 2 ? 0 : -1;
}
static inline int kv_read_u32(uint16_t key, uint32_t *out) {
    return kv_read(key, (uint8_t *)out, 4) == 4 ? 0 : -1;
}
static inline int kv_read_string(uint16_t key, char *buf, uint16_t buf_size) {
    int len = kv_read(key, (uint8_t *)buf, buf_size - 1);
    if (len > 0) buf[len] = '\0';
    return len;
}

// Write/update an APP-owned KV entry. Returns 0 on success.
// BL-owned keys (0x0001-0x00FF) are rejected with -EPERM.
int kv_write(uint16_t key, const uint8_t *value, uint16_t len);

// Delete an APP-owned KV entry (marks tombstone).
int kv_delete(uint16_t key);

// Compact APP region: erase sector, rewrite BL + valid APP entries.
// Called when APP region is full or on explicit request.
int kv_compact(void);
```

#### 4.3.5 Application Region Persistence

The KV area is erased each boot so the bootloader can write fresh entries. Application config must survive boots. Two strategies:

**Strategy A — Separate APP KV sector (recommended for production):**

Allocate a second 4 KB sector at 0x22_2000 for application-persistent KV:

```
0x22_1000   4 KB   BL KV Area     Erased + rewritten each boot by bootloader
0x22_2000   4 KB   APP KV Area    Never erased by bootloader. Application manages.
0x22_3000   ~1.863 MB Reserved
```

The bootloader never touches the APP KV sector. The application links `libsbop-kv` which manages the APP KV area independently.

**Strategy B — Save/Restore (memory-constrained, single sector):**

Bootloader reads APP entries before erase, saves to DTCM, erases sector, writes BL entries, restores APP entries. Requires ~2 KB DTCM buffer.

```c
// In bootloader Phase 11, before KV area erase:
uint8_t app_backup[2048];
uint16_t app_backup_len = kv_read_app_region_raw(app_backup, sizeof(app_backup));
kv_erase_sector(KV_AREA_ADDR);
kv_write_bl_entries();  // Populate BL region
kv_restore_app_region(app_backup, app_backup_len);  // Restore APP entries
```

**Recommendation for STM32H750VBT:** Strategy A (separate sectors). The 4 MB NOR flash has ample space; a second 4 KB sector costs 0.1% of total capacity. Separation eliminates the backup/restore race condition and keeps bootloader KV logic simple.

Updated NOR flash layout with Strategy A:

```
0x22_1000   4 KB   BL KV Area      Erased + rewritten each boot by bootloader
0x22_2000   4 KB   APP KV Area     Application-persistent config (never touched by BL)
0x22_3000   ~1.863 MB Reserved
```

#### 4.3.6 Security Properties

| Property | Mechanism |
|----------|-----------|
| BL entries are read-only to app | `flags & 0x02` enforced by `kv_write()` — keys < 0x0100 rejected with -EPERM |
| APP entries are isolated from BL | Separate sector: BL never writes APP sector |
| Entry integrity | CRC-16 per entry. Corrupt entries skipped on read with error logged |
| Rollback of APP config | KV area outside slot images — rollback does not affect app config |
| No information leak across boots | BL KV sector fully erased each boot |
| Write protection | Both KV sectors in BP-protected NOR region after LOCK_BOOT; APP KV writes go through SVC that temporarily unlocks BP |

#### 4.3.7 Board Information Area vs KV Storage Area

| Aspect | Board Information Area (0x22_0000) | KV Storage Area (0x22_1000/0x22_2000) |
|--------|-----------------------------------|---------------------------------------|
| Purpose | Factory-provisioned device identity | Runtime boot→app handoff + app config |
| Written by | Provisioning station (factory) | Bootloader (each boot) + Application |
| Write frequency | Once at factory, rarely in field | BL region: every boot. APP region: as needed |
| Format | Fixed struct with CRC-32 | TLV entries with CRC-16 each |
| Content | SKU, serials, factory config | Boot data, FW version, app settings |
| Protected by | BP bits after LOCK_BOOT | BP bits after LOCK_BOOT |
| Erased each boot | No | BL sector: yes. APP sector: no |

#### 4.3.8 Size Budget

| Component | Size |
|-----------|------|
| KV Area Header | 16 bytes |
| BL region (17 well-known entries) | ~400 bytes (7-byte header + avg 15-byte value + 2-byte CRC each) |
| BL region overhead (deleted tombstones after N boots) | 0 (erased fresh each boot) |
| APP region capacity | 2048 bytes (~100 typical small KV entries) |
| **BL KV sector total** | **4 KB** |
| **APP KV sector total** | **4 KB** |
| **Combined KV footprint** | **8 KB (0.2% of 4 MB NOR)** |

**libsbop-kv library size (application):** ~2 KB code (linear scan, CRC-16, erase/write). Linked into application or provided as part of libsbop-ota.

---

## 5. SBOP Requirement Coverage Assessment

### 5.1 Boot Security

| SBOP Requirement | Implementation | Status |
|---|---|---|
| REQ-FR-BOOT-001 | Ed25519 signature verification (fiat + tinycrypt-sha512). 400 MHz Cortex-M7: ~1.2 ms verify. Formal verification (Coq) for field arithmetic. FIH-protected redundant check. **AES-256-CTR decrypt on-the-fly to AXI SRAM during Phase 5 — plaintext payload never in NOR flash.** | **Pass** |
| REQ-FR-BOOT-002 | SHA-256 via hardware HASH accelerator. ~13 ms for 512 KB image (hardware HASH). FIH redundant hash check with header re-read. | **Pass** |
| REQ-FR-BOOT-003 | 12-phase state machine in bootloader .text. Phase tracking variable with CRC (ERR-BOOT-STATE-002 on mismatch). | **Pass** |
| REQ-FR-BOOT-004 | Version counter in NOR SR1 (4× redundant 32-bit copies). FIH redundant read-compare (6 independent comparisons). Anti-rollback: version increment capped at +1 per boot, minimum allowed version floor in SR1, 4-way redundant counter with mismatch detection. | **Pass** |
| REQ-FR-BOOT-005 | FAILSAFE state: LED/GPIO indicator, watchdog timeout, Y-Modem recovery via USART1. Two-copy metadata journal survives power loss. NOR BP bits hardware-protect metadata. | **Pass** |
| REQ-FR-BOOT-006 | Boot health reporting (N=3 attempt policy). Application writes health status early in init. Bootloader rolls back only after 3 consecutive failures without health report. Prevents false rollback on transient faults. | **Pass** |
| REQ-FR-BOOT-007 | PC-ROP execute-only sectors for KD code + no MPU (temporal isolation). Bootloader clears sensitive state before EXECUTE jump. NOR flash write-protect BP bits set before jump. | **Pass** |
| REQ-FR-BOOT-008 | Automatic slot fallback: bootloader attempts fallback slot before entering FAILSAFE. Both slots must be dead for FAILSAFE. Only 2 attempts per boot (bounded). Slot marked INVALID on verification failure, fallback gets fresh boot health window. | **Pass** |
| REQ-FR-BOOT-009 | IWDG hardware watchdog (800 ms timeout, LSI ~32 kHz, prescaler=32, reload=820). Two-pass boot: Pass 1 runs full verification with IWDG protection, then sets RTC BKP0R magic + NVIC_SystemReset. Pass 2 detects RTC magic, skips IWDG init, jumps directly to app (~5 ms). Application starts with IWDG in hardware reset state (disabled). NOR presence verified via JEDEC ID (9FH: Boya 0x68 or Winbond 0xEF) before any SPI read. ±5% LSI drift over -40°C to +85°C: worst-case margin 588 ms (>3× boot budget). | **Pass** |

### 5.2 OTA / Update

OTA is implemented as a **service library** (`libsbop-ota`) linked into the application (Zone 2). The bootloader is not involved in OTA operations — it only verifies and executes images already written to slots.

| SBOP Requirement | Implementation | Status |
|---|---|---|
| REQ-FR-OTA-001 | **Application OTA library:** TLS 1.3 + backend auth. OTA verifies backend response signature (BACKEND_AUTH) + manifest signature using embedded public key from ManifestSignatureBlock (in-band CSM multi-key) + encrypted blob hash. **No decryption in OTA** — plaintext never written to NOR. Image stays encrypted (AES-256-CTR) in NOR at all times. Bootloader decrypts on-the-fly to AXI SRAM during Phase 5. Key fingerprint + revocation enforced by Gate 2 (bootloader, Zone 1). | **Pass** |
| REQ-FR-OTA-003 | **Application OTA library:** SHA-256 of encrypted blob verified against signed manifest hash. NOR write verified by read-back + hash re-check. InnerSignatureBlock (Ed25519) stored alongside ciphertext for boot-time verification. | **Pass** |
| REQ-FR-OTA-004 | Dual A/B slots in NOR flash (1 MB each). Slot metadata (status, version, CRC) in dedicated 128 KB sector. Atomic slot transition via metadata write → CRC verify → commit. | **Pass** |
| REQ-FR-OTA-005 | **Bootloader:** detects ACTIVE slot failure at boot → marks slot INVALID → switches to FALLBACK → increments rollback_count in NOR SR1. **Application OTA library:** ota_rollback() for application-initiated rollback. | **Pass** |
| REQ-SR-OTA-ENC | X25519 hybrid encryption: backend ECDH-encrypts once with OTA_public[i] → K_s = HKDF-SHA-256(X25519(e, OTA_public[i])). AES-256-CTR encrypts Payload only. Ephemeral key E + IV plaintext in NOR. InnerSignatureBlock outside encryption envelope. **OTA never decrypts** — encrypted blob written to NOR as-is. Bootloader ECDH-decrypts on-the-fly to AXI SRAM (volatile). Re-encrypts with KD (per-device at-rest). Plaintext never persists in non-volatile storage. K_s in CRYP registers only, zeroized after use. | **Pass** |

### 5.3 Identity & Anti-Cloning

| SBOP Requirement | Implementation | Status |
|---|---|---|
| REQ-SR-ID-001 | Composite UID = MCU 96-bit UID + NOR 128-bit UID (224 bits). KD derived from KR + composite UID. | **Pass** |
| REQ-SR-ID-002 | NOR-MCU binding in SR3 (OTP). KD commitment in SR2 (OTP). Backend registration with composite UID dedup. | **Pass** |
| ERR-ID-CLONE-001 | Binding verification at boot. Mismatch → FAILSAFE. | **Pass** |
| REQ-SR-ID-003 | Board Information Area (NOR 0x22_0000): board_id (SKU), board_name, board_serial, system_name, system_serial, config_flags, config_data. CRC-32 protected. Written at factory provisioning, config updatable via authenticated SVC. BP-bits protected after boot. | **Pass** |
| ERR-BOOT-PARSE-008 | ImageHeader.board_id checked against stored board_id in Phase 4 (FIH redundant). Mismatch → FAILSAFE. Prevents cross-SKU firmware loading. | **Pass** |

### 5.4 Cryptographic

| SBOP Requirement | Implementation | Status |
|---|---|---|
| REQ-SR-BOOT-001 | Ed25519 (fiat-curve25519 + tinycrypt-sha512) — formally verified, constant-time. NXP CSM-style multi-key (4 key pairs): public keys embedded in image SignatureBlock, verified against SR1 OTP fingerprints at boot. Key revocation via OTP bitmap. Single-key legacy fallback (key_index=0xFF) uses PC-ROP .rodata key. | **Pass** |
| REQ-SR-BOOT-002 | SHA-256 via STM32 HASH hardware. HMAC-SHA-256 for KD commitment, NOR-MCU binding. | **Pass** |
| REQ-SR-BOOT-003 | AES-256-CTR via STM32 CRYP hardware for image decryption. X25519 ECDH with OTA_private[i] + ephemeral E → K_s = HKDF-SHA-256(S, "SBOP-OTA-ENC") for first boot. KD direct decrypt for subsequent boots. | **Pass** |
| TRNG health tests | STM32 RNG with built-in entropy source validation. Bootloader runs additional FIPS 140-2 continuous test at INIT phase. | **Pass** |

### 5.5 Physical Security

| SBOP Requirement | Implementation | Status |
|---|---|---|
| REQ-SR-PHY-001 | 3× TAMP pins for enclosure tamper detection. FIH redundant checks for fault injection. CRC-32 on critical state variables. | **Pass** |
| REQ-SR-PHY-002 | Tamper interrupt → key material zeroization (DTCM, backup SRAM, CRYP key registers) → enter FAILSAFE. CRYP K0LR–K3RR explicitly cleared via HAL_CRYP_DeInit(). Backup SRAM survives reset for tamper event logging. IWDG reset → automatic retry of alternate slot. | **Pass** |
| Fault injection defense | FIH pattern (MCUboot-derived): no early-return, FIH_INT accumulator, redundant verification, multi-bit FIH_SUCCESS/FIH_FAILURE. | **Pass (software)** |

### 5.6 Debug Security

| SBOP Requirement | Implementation | Status |
|---|---|---|
| REQ-SR-DBG-001 | ROP Level 1 after provisioning. JTAG/SWD flash read disabled. | **Pass** |
| REQ-SR-DBG-002 | Challenge-response debug unlock via bootloader KD_Debug. Not hardware-enforced. | **Partial** |

---

## 6. Gap Analysis

| # | Gap | Severity | Mitigation | SL Impact |
|---|---|---|---|---|
| G1 | **No hardware secure element.** Keys stored in PC-ROP flash; KD in RAM only during boot. | Medium | KD zeroization on tamper/FAILSAFE. Ed25519 public key only (no private key on device). For SL3+: add ATECC608 on I2C. | SL2 acceptable; SL3 needs SE |
| G2 | **No hardware ECDSA/Ed25519 accelerator.** Software Ed25519 at 400 MHz is fast (~1.2 ms verify) but side-channel resistance depends on implementation quality. | Low | Use fiat-curve25519 + tinycrypt-sha512 (formally verified, constant-time). Avoid ECDSA (variable-time unless carefully implemented). | SL2 acceptable |
| G3 | **No dedicated glitch sensor IC.** FIH provides software-level glitch resistance. POR/BOR gives basic voltage monitoring. | Medium | External voltage supervisor (e.g., MAX16151) on VCC for SL3+. FIH redundant checks catch logical glitch effects. | SL2 acceptable; SL3 needs external IC |
| G4 | **External NOR flash physically accessible.** Firmware images can be extracted via PCB probing. | Medium | X25519 ECDH hybrid encryption (4 OTA key pairs, same for all devices). Backend encrypts once with OTA_public[i]; bootloader ECDH-decrypts to AXI SRAM; re-encrypts with KD (per-device at-rest). NOR flash stores ciphertext only. Plaintext exists only in volatile AXI SRAM during ~300 ms boot window. | Mitigated |
| G5 | **NOR flash Security Registers vendor-dependent size.** BY25Q32ES: 3× 1024 bytes. Winbond W25Q32JV: 3× 256 bytes. | Low | Design caps SR usage at 256 bytes per register (~96 bytes currently used). Cross-vendor compatible. Once OTP-locked, no further writes. | Acceptable |
| G6 | **Standard SPI at 80 MHz (operational).** NOR flash supports 120 MHz max; configured at 80 MHz (SPI1 PCLK 160 MHz ÷ 2) for timing margin. Quad SPI would give 4× bandwidth but adds pin count. | Low | 80 MHz SPI ≈ 10 MB/s. 512 KB image reads in ~51 ms. Full boot (read + verify + copy) < 300 ms achievable. | Acceptable |
| G7 | **No authenticated debug hardware.** KD_Debug challenge-response is software-only, no hardware lockout enforcement. | Low | ROP Level 2 for permanent debug disable (irreversible fuse). For SL3+: external secure element can gate debug access. | SL2 acceptable |

---

## 7. Security Level Assessment

| Security Function | SL-T Target | Achievable | Notes |
|---|---|---|---|
| Boot integrity | SL-T 2 | **SL-T 2** | Ed25519 + SHA-256 + FIH + IWDG + slot fallback |
| OTA integrity | SL-T 2 | **SL-T 2** | Dual-slot + atomic metadata |
| Anti-rollback | SL-T 2 | **SL-T 2** | OTP version counter |
| Identity / Anti-cloning | SL-T 2 | **SL-T 2** | Composite UID + OTP binding |
| Key protection | SL-T 2 | **SL-T 2** | PC-ROP + key zeroization |
| Physical tamper | SL-T 2 | **SL-T 2** | 3× TAMP pins + FIH |
| Side-channel | SL-T 2 | **SL-T 2** | Ed25519 fiat (formally verified, constant-time) |
| Debug security | SL-T 2 | **SL-T 2** | ROP Level 1/2 |
| Supply chain | SL-T 2 | **SL-T 2** | OTP lock + backend registration |

**Overall: SL-T 2 achievable on this platform as-is.** All 9 security functions assessed at SL-T 2 with the defined architecture — Ed25519 software verify, FIH glitch detection, per-device encryption, OTP binding, and temporal isolation. No hardware secure element, no dedicated glitch sensor IC, and no MPU — each gap has an acceptable software-level mitigation for SL-T 2.

**SL-T 3 path:** Add ATECC608 (secure element) + MAX16151 (glitch supervisor) + EM shield over NOR flash.

---

## 8. Bootloader Size Budget

| Component | Estimated Size | Notes |
|---|---|---|
| Vector table + startup | 512 B | Fixed |
| State machine + flow control | 2 KB | 12-phase state machine with phase tracking |
| SPI NOR flash driver | 2 KB | Standard SPI, read/erase/security register (80 MHz) |
| Image header parser + validator | 2 KB | Fixed-format parser, no allocation |
| Ed25519 verify (fiat + tinycrypt-sha512) | 6 KB | fiat-curve25519 (~2.5 KB) + verify logic (~2 KB) + tinycrypt SHA-512 (~1.5 KB). Formally verified Coq. Raw r||s, no ASN.1.|
| SHA-256 (HAL + wrapper) | 2 KB | Hardware HASH via HAL |
| FIH infrastructure | 1 KB | fih_int, fih_eq, fih_dec, redundant checks |
| AES-CTR decrypt + re-encrypt + AXI copy | 1.5 KB | ECDH decrypt (K_s) → verify → KD re-encrypt → NOR write-back → AXI SRAM copy |
| Y-Modem Rx (recovery) | 2 KB | Polled USART1, 1K CRC, frame parser, retry |
| USART1 driver (polled) | 0.5 KB | Recovery console, 115200–921600 bps |
| Recovery slot selector | 0.5 KB | Finds first EMPTY or INVALID slot for Y-Modem |
| Error handler + FAILSAFE | 2 KB | Error codes, logging, tamper response |
| Slot fallback logic | 0.35 KB | Automatic fallback before FAILSAFE (§3.12) |
| IWDG watchdog + NOR detect + RTC magic | 0.2 KB | IWDG init + refresh + JEDEC ID check + two-pass boot (§3.13) |
| KD access + HKDF + X25519 + SVC handler + commitment | 4 KB | PC-ROP: KD derive, X25519 ECDH, KD_Auth, CRYP key load, SVC dispatch |
| Metadata journal (two-copy atomic) | 0.3 KB | Sequence-based journal, CRC verify, rollback on partial write |
| Boot health reporting | 0.2 KB | N-attempt policy (N=3) with prev_boot_phase tracking |
| Anti-rollback (4-way redundant) | 0.2 KB | Redundant SR1 reads, min version check, capped increment |
| .rodata (keys, refs, strings) | 1 KB | Error strings, key references, SR1 offset constants. Multi-key mode: no public keys in .rodata. Legacy mode: 32 B KI_public. |
| .data + .bss + stack | 12 KB | Runtime state, 4 KB stack (DTCM) |
| Option bytes + config | 2 KB | Reserved, not in code budget |
| Key fingerprint verify + revocation check | 0.5 KB | SHA-256(fingerprint) compare + SR1 revocation bitmap read (CSM multi-key) |
| **Total .text + .rodata + .data** | **~32.05 KB** | Bootloader — decrypt, verify, re-encrypt, slot fallback, IWDG two-pass. No OTA/TLS/HTTP. |
| **Total with stack** | **~44.05 KB** | Well within 128 KB available (34% utilization) |

**Application OTA library (libsbop-ota, separate budget in Zone 2):**

| Component | Estimated Size | Notes |
|---|---|---|
| OTA state machine + flow | 2 KB | AUTHENTICATE → CHECK → DOWNLOAD → VERIFY → INSTALL → ACTIVATE |
| Ed25519 verify (fiat, shared with bootloader) | (in bootloader .text) | Signature verification — OTA uses embedded public key from ManifestSignatureBlock (in-band CSM multi-key). Key fingerprint + revocation enforced by Gate 2 bootloader. |
| SHA-256 (HW HASH + wrapper) | 0.5 KB | Encrypted blob hash verification |
| HTTP Range download + resume | 4 KB | Chunked download with checkpoint |
| TLS 1.3 + cert verify | ~50 KB | mbedTLS or wolfSSL, shared with app |
| Backend auth protocol | 2 KB | Challenge-response with KD_Auth |
| Image write + verify | 1 KB | SPI NOR write ciphertext, read-back, hash check (no decrypt) |
| Slot metadata update | 1 KB | Atomic slot status transitions |
| **Total libsbop-ota** | **~12 KB** | Excluding TLS stack (already in app). No AES-CTR/decrypt in OTA library. |

---

## 9. References

| Document | Reference |
|---|---|
| STM32H750 Reference Manual | RM0433 Rev 8 |
| STM32H750xB Datasheet | DS12556 Rev 8 |
| BY25Q32ES Datasheet | QA-DT-006 Rev 2.3 (JEDEC ID: 0x68 0x40 0x16) |
| Winbond W25Q32JV Datasheet | Cross-vendor reference (JEDEC ID: 0xEF 0x40 0x16, 3× 256 B SRs) |
| MCUboot FIH Design | https://www.trustedfirmware.org/projects/mcuboot/ |
| SBOP Boot Overview | `../../03_Subsystem/Boot/Boot_Overview.md` |
| SBOP Error Code Catalog | `../../02_System_Design/Error_Code_Catalog.md` |
| SBOP Security Goals | `../../04_Security/Security_Goals.md` |
| SBOP Key Hierarchy | `../../02_System_Design/Key_Hierarchy.md` |
| SBOP Attack Tree | `../../04_Security/Attack_Tree.md` |
| SBOP Mitigation Strategy | `../../04_Security/Mitigation_Strategy.md` |
