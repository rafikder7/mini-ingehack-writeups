---
title: keep-secrets
category: rev
difficulty: medium
points: 100-500
author: RedhaIWNL
date: 2026-11-01
---
# keep-secrets

**Category:** rev · **Difficulty:** medium · **Flag:** `ingehack{zero_keeps_his_secrets}`

Lelouch keeps a small ledger for the Black Knights. The app shows five entries,
but every one is ciphertext. One of them is the flag. The app never decrypts,
never talks to a server, and never writes the plaintext into the APK — the
ciphertext and the sealing code are the whole challenge.

## Handout

- `files/keep-secrets.apk` — signed release APK, no runtime permissions.
- Package: `com.ingehack.keepsecrets`
- Min SDK 24; built against SDK 36.
- The app is a Kotlin + Jetpack Compose vault. The background is Code Geass
  fan art supplied by the challenge author for flavor; it is not part of the
  puzzle.

## Intended solve

### 1. Look at the app

Install the APK and open it. The five sealed entries are **Ashford**,
**Nunnally**, **Black Knights**, **???**, and **C.C.**. Their Base64 blobs are
generated at build time, so the exact strings in your copy will differ; read
them from `SecretData` or from the app.

The **NEW ENTRY** box lets you seal your own text, but there is no reveal
button. The app only calls `VaultCrypto.seal(...)`.

### 2. Decompile with jadx

```bash
jadx-gui files/keep-secrets.apk
```

Open `com.ingehack.keepsecrets.VaultCrypto`. The relevant code is:

```kotlin
object VaultCrypto {
    private const val FIRST = "All hail"
    private const val SECOND = "Lelouch"
    private val PASSPHRASE = "$FIRST $SECOND"
    private val KEY = MessageDigest.getInstance("SHA-256")
        .digest(PASSPHRASE.toByteArray(Charsets.UTF_8))

    fun seal(plaintext: String): String {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, SecretKeySpec(KEY, "AES"))
        val iv = cipher.iv
        val ciphertext = cipher.doFinal(plaintext.toByteArray(Charsets.UTF_8))
        return Base64.encodeToString(iv + ciphertext, Base64.NO_WRAP)
    }
}
```

So:

1. The passphrase is `All hail Lelouch`.
2. The AES-256 key is `SHA-256("All hail Lelouch")`.
3. Every blob is `Base64(12-byte IV || AES-GCM ciphertext || 16-byte GCM tag)`.
4. The GCM tag is part of the ciphertext, so a normal AES-GCM decrypt with the
   first 12 bytes as the IV works directly.

`SecretData` in the same package contains the five Base64 blobs.

### 3. Decrypt

The short version, with `cryptography` installed:

```python
import base64
import hashlib
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

key = hashlib.sha256(b"All hail Lelouch").digest()
blob = base64.b64decode("...")          # one value from SecretData
raw = blob
iv, ciphertext_and_tag = raw[:12], raw[12:]
print(AESGCM(key).decrypt(iv, ciphertext_and_tag, None).decode())
```

Or automate the whole thing — decompile the two app classes, recover the
passphrase, and decrypt every entry:

```bash
python3 solution/solve.py files/keep-secrets.apk
```

Expected output:

```text
passphrase: 'All hail Lelouch'
key (SHA-256): 38a68f2277568ac9658306f1cf42daa1806bbbf23620689d80453f80395024d5
Ashford: The student council room is safe. For now.
Nunnally: I will build a gentler world for her.
Black Knights: The next operation begins at midnight.
???: ingehack{zero_keeps_his_secrets}
C.C.: She still owes me a pizza.
```

The `???` entry is the flag.

## Why this is medium

- The app is a real Kotlin/Compose APK, not a strings-only handout. `jadx`
  shows the crypto object, the key derivation, and the cipher mode.
- The key is not a raw byte array or a plaintext string assigned to
  `SecretKeySpec`. The player has to notice the passphrase split, hash it with
  SHA-256, and use AES-GCM rather than CBC.
- The IV is prepended and the GCM tag is appended by the JCE; the player has
  to parse the blob format correctly.
- No server, no container, no dynamic flag. The challenge is entirely local.

## Rebuilding

The source is in `solution/app/`. The APK is generated from a clean build:

```bash
cd solution
./build.sh
```

`build.sh` runs `gen_secrets.py` (which regenerates `SecretData.kt` from the
passphrase in `VaultCrypto.kt` and the flag in `chall.yaml`), builds the release
APK with the Gradle wrapper, aligns it, and signs it with the local throwaway
key in `solution/app/keystore.jks`. The keystore is intentionally untracked.

The app has no runtime permissions and no network code. `minifyEnabled` is off
so the crypto class remains readable — that is deliberate for a reversing
challenge, and it is why the flag must stay out of the APK in plaintext.

## Notes

- The plaintext flag exists only in `chall.yaml` and in this solution. The
  generator encrypts it before writing `SecretData.kt`.
- The APK is signed and zipaligned; install with
  `adb install -r files/keep-secrets.apk`.
- The background image is fan art supplied by the author for this private CTF
  challenge. Keep the credit if the challenge is reused publicly.
