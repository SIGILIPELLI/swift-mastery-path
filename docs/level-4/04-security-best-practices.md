# 04 · Security Best Practices

Swift ships `CryptoKit`, a modern, misuse-resistant cryptography API, as
part of the standard toolchain — no external dependency needed. This
module covers hashing, message authentication, symmetric encryption, and a
few non-cryptographic habits (constant-time comparison, input validation)
that matter just as much in practice.

## Hashing with SHA-256

Hashing is one-way: it produces a fixed-size fingerprint of data, useful
for integrity checks, but never for storing passwords directly (see below):

```swift
import CryptoKit
import Foundation

let password = "correct horse battery staple"
let digest = SHA256.hash(data: Data(password.utf8))
let hex = digest.map { String(format: "%02x", $0) }.joined()
print("SHA256:", hex)
```

Output:

```
SHA256: c4bbcb1fbec99d65bf59d85c8cb62ee2db963f0fe106f483d9afa73bd4e39a8a
```

**Never hash passwords with a plain, fast hash like SHA-256 for storage.**
Password storage needs a slow, salted, purpose-built algorithm (bcrypt,
scrypt, or Argon2) specifically because it must resist brute-force
guessing — SHA-256 is *fast*, which is exactly the wrong property for that
job. `SHA256` here is for data-integrity fingerprints, not credentials.

## HMAC: verifying a message hasn't been tampered with

An HMAC combines a hash function with a secret key, proving both that data
is unmodified *and* that it was produced by someone who knows the key:

```swift
let key = SymmetricKey(size: .bits256)
let message = "transfer $100 to Bob".data(using: .utf8)!
let hmac = HMAC<SHA256>.authenticationCode(for: message, using: key)
print("HMAC verifies:", HMAC<SHA256>.isValidAuthenticationCode(hmac, authenticating: message, using: key))
```

Output:

```
HMAC verifies: true
```

This is the mechanism behind signed URLs, webhook signature verification,
and API request signing — the sender computes an HMAC over the payload with
a shared secret, and the receiver recomputes it independently to confirm
nothing changed in transit.

## Symmetric encryption with AES-GCM

`AES.GCM` is authenticated encryption — it protects both confidentiality
(the data is unreadable without the key) and integrity (tampering is
detected) in one primitive:

```swift
let secretKey = SymmetricKey(size: .bits256)
let plaintext = "sensitive data".data(using: .utf8)!
let sealedBox = try! AES.GCM.seal(plaintext, using: secretKey)
let ciphertext = sealedBox.combined!
print("Encrypted length:", ciphertext.count, "original length:", plaintext.count)

let opened = try! AES.GCM.open(AES.GCM.SealedBox(combined: ciphertext), using: secretKey)
print("Decrypted:", String(data: opened, encoding: .utf8)!)
```

Output:

```
Encrypted length: 42 original length: 14
Decrypted: sensitive data
```

The ciphertext is longer than the plaintext because `combined` bundles the
nonce and the authentication tag alongside the actual encrypted bytes —
all three are needed to decrypt and verify, so `combined` is the form you
should store or transmit as a single unit.

## Constant-time comparison

Comparing secrets (tokens, MACs) with `==` can leak timing information —
`==` on strings typically short-circuits at the first mismatched byte,
and an attacker measuring response times can exploit that to guess a
secret one byte at a time:

```swift
func constantTimeEquals(_ a: String, _ b: String) -> Bool {
    let aBytes = Array(a.utf8)
    let bBytes = Array(b.utf8)
    guard aBytes.count == bBytes.count else { return false }
    var diff: UInt8 = 0
    for i in 0..<aBytes.count { diff |= aBytes[i] ^ bBytes[i] }
    return diff == 0
}
print("Constant-time compare (equal):", constantTimeEquals("secret-token", "secret-token"))
print("Constant-time compare (different):", constantTimeEquals("secret-token", "wrong-token!"))
```

Output:

```
Constant-time compare (equal): true
Constant-time compare (different): false
```

The loop always runs over every byte regardless of where a mismatch
occurs, so its execution time doesn't reveal *where* the strings first
differ — `CryptoKit`'s own `HMAC.isValidAuthenticationCode` above already
does this internally, so prefer it over hand-rolled comparisons whenever a
`CryptoKit` API is verifying its own output.

## Input validation

Never trust data crossing a trust boundary (user input, request bodies,
query parameters) without validating its shape first:

```swift
func isValidEmail(_ email: String) -> Bool {
    let pattern = #"^[^\s@]+@[^\s@]+\.[^\s@]+$"#
    return email.range(of: pattern, options: .regularExpression) != nil
}
print("Valid email:", isValidEmail("user@example.com"))
print("Invalid email:", isValidEmail("not-an-email"))
```

Output:

```
Valid email: true
Invalid email: false
```

This pattern is intentionally loose — real email validation is
famously hard to get exactly right with a single regex — but rejecting
obviously malformed input early, before it reaches a database query or an
external API call, closes off a large class of injection and malformed-data
bugs cheaply.

## Storing secrets: Keychain, not `UserDefaults`

On Apple platforms, `UserDefaults` is a plist file with no encryption at
rest — appropriate for preferences, never for tokens, passwords, or API
keys. The Keychain (via `Security.framework`, or a wrapper like
`KeychainAccess`) is the correct store, encrypted and access-controlled by
the OS. Keychain APIs need a full Xcode/simulator or device environment to
exercise meaningfully, so they aren't demonstrated with runnable output
here — but the rule (Keychain for secrets, `UserDefaults` for everything
else) applies regardless of platform availability.

## Swift-specific traps

- **`try!` on cryptographic operations (as used above for brevity) is a
  real crash risk in production** — `AES.GCM.seal`/`.open` can throw (a
  corrupted or tampered ciphertext, a wrong key), and production code
  should handle that with `try`/`catch`, surfacing a generic
  "decryption failed" rather than crashing or leaking why it failed.
- **`SymmetricKey(size:)` generates a new random key every time** — it is
  not deterministic and not derived from any string you might expect;
  deriving a key from a password requires a key-derivation function like
  HKDF (also in `CryptoKit`), not a raw `SymmetricKey` initializer.
- **A `SealedBox`'s `combined` property is optional** (`nil` in some
  non-default nonce configurations) — force-unwrapping it, as this module
  does for brevity, should become a proper `guard let` in real code.
- **Regex-based validation is a first filter, not a security boundary on
  its own** — always pair input validation with parameterized queries
  (Module 03, Level 3) and proper output encoding; validation alone doesn't
  prevent injection if the validated string still reaches an unsafe sink.

## Cheat sheet

| Need | `CryptoKit` API |
|------|-----------------|
| Data integrity fingerprint | `SHA256.hash(data:)` |
| Verify a message + shared secret | `HMAC<SHA256>.authenticationCode` / `.isValidAuthenticationCode` |
| Encrypt + authenticate data | `AES.GCM.seal` / `.open` |
| Compare secrets safely | `HMAC.isValidAuthenticationCode`, or a manual constant-time XOR loop |
| Store a secret on-device | Keychain (never `UserDefaults`) |

## Exercise

Write a function `func sign(payload: String, secret: SymmetricKey) ->
String` that computes an HMAC-SHA256 over the payload and returns it
hex-encoded (reuse the hex-encoding pattern from the SHA-256 example).
Write a matching `func verify(payload: String, signature: String, secret:
SymmetricKey) -> Bool` that recomputes the signature and compares it to the
provided one using `HMAC.isValidAuthenticationCode` (not a plain `==` on
the hex strings). Demonstrate that tampering with even one character of
`payload` after signing causes `verify` to return `false`.
