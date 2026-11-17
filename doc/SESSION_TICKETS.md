# Session Tickets on Embedded Targets

wolfSSL session tickets let a client resume a TLS session without the server
keeping per-client session state in RAM. The ticket is an encrypted blob that
the client stores and presents on a later connection. This is the resumption
mechanism used by TLS 1.3; TLS 1.2 servers can use the same ticket key context.

## Build configuration

Session tickets need `HAVE_SESSION_TICKET`. With Autotools:

```sh
./configure --enable-session-ticket --enable-tls13
```

With `user_settings.h` (the usual embedded or IDE build), define it in
`user_settings.h` and define `WOLFSSL_USER_SETTINGS` for both the library and
the application:

```c
#define WOLFSSL_USER_SETTINGS
#define HAVE_SESSION_TICKET
#define WOLFSSL_TLS13
```

`examples/configs/user_settings_embedded.h` exposes this as
`WC_CFG_SESSION_TICKET`. Enabling it also requires `WC_CFG_RTC`: ticket
lifetimes are time based, and `wolfssl/wolfcrypt/settings.h` removes ticket and
session-cache support when `NO_ASN_TIME` is set.

A server also needs certificate and private-key support unless it uses PSK
only. The examples below assume a normal certificate-authenticated server.

## Built-in ticket encryption

When `WOLFSSL_NO_DEF_TICKET_ENC_CB` is not defined, a server context installs
the default ticket encryption callback. The cipher is selected at build time:

- `WOLFSSL_TICKET_ENC_CBC_HMAC` selects AES-CBC plus HMAC.
- Otherwise ChaCha20-Poly1305 is used when `HAVE_CHACHA` and `HAVE_POLY1305`
  are available and no AES-GCM variant was explicitly selected.
- Otherwise AES-GCM is used: AES-256-GCM when `WOLFSSL_AES_256` is available,
  or AES-128-GCM when it is not.
- SM4-GCM is the fallback when neither ChaCha20-Poly1305 nor AES-GCM is
  available and `WOLFSSL_SM4_GCM` is defined.

The key name, IV and ticket length are authenticated as additional data. The
last bit of the 16-byte key name identifies which of the two key slots
encrypted the ticket. Do not use a custom callback with the built-in rotation
API unless it implements the same contract.

## Automatic and explicit rotation

The default callback keeps two keys, referred to as the current and previous
key:

- New tickets are issued with the current key.
- A key is used for encryption only while the ticket lifetime hint fits inside
  its remaining key lifetime.
- Once the current key is too close to expiry, the other key becomes current.
- A new key is generated when neither key can cover the ticket lifetime.
- A previous key remains usable for decryption until its own expiry.

This automatic rotation requires no application work. `WOLFSSL_TICKET_KEY_LIFETIME`
sets the key lifetime in seconds and is one hour by default. It must be greater
than `SESSION_TICKET_HINT_DEFAULT` (300 seconds).

An application that has its own rotation policy can force a rotation:

```c
if (wolfSSL_CTX_rotate_ticket_keys(server_ctx) != WOLFSSL_SUCCESS) {
    /* handle RNG or state error */
}
```

The call generates a new key into the non-current slot and makes it current.
The key that was current before the call remains available for decryption, so
tickets issued just before the rotation can still resume. The replacement
ticket sent after that recovery is encrypted with the new key.

Do not rotate more often than the ticket lifetime hint. A third rotation
reuses the oldest slot, so a ticket older than the previous generation can no
longer be decrypted.

`wolfSSL_CTX_rotate_ticket_keys()` only operates on the built-in callback. It
returns `BAD_STATE_E` when `wolfSSL_CTX_set_TicketEncCb()` has installed a
custom callback. For a custom callback, implement rotation in the callback's
user context.

## TLS 1.3 behavior

TLS 1.3 issues `NewSessionTicket` messages after the handshake, and can issue
more than one. Each ticket uses the current encryption key at the time it is
sent. On resumption:

1. The client presents the ticket in its `pre_shared_key` extension.
2. The server decrypts it with the key named in the ticket.
3. A valid ticket resumes the session even if it was issued under the
   previous key.
4. The server sends a new `NewSessionTicket`, encrypted with the current key.

TLS 1.2 uses the same key context for RFC 5077 tickets. A ticket decrypted
under the previous key can still resume and, when the callback requests
renewal, the server can issue a replacement ticket under the current key.

## Persisting keys across restarts

By default the keys are generated lazily and live only in the `WOLFSSL_CTX`.
Tickets don't survive a restart unless the application saves and restores the
key material. Get the current serialized key set after the context is created:

```c
byte keys[WOLFSSL_TICKET_KEYS_SZ];

if (wolfSSL_CTX_get_tlsext_ticket_keys(server_ctx, keys, sizeof(keys)) !=
        WOLFSSL_SUCCESS) {
    /* handle error */
}
/* Store keys in protected persistent storage. */
```

On the next boot, restore the keys before creating any `WOLFSSL` object or
accepting connections:

```c
if (wolfSSL_CTX_set_tlsext_ticket_keys(server_ctx, keys, sizeof(keys)) !=
        WOLFSSL_SUCCESS) {
    /* handle error */
}
```

The serialized blob contains the key name, both encryption keys and both
expiry timestamps. It is secret key material:

- Encrypt or protect it with the platform's secure storage.
- Never log it or put it in a firmware image shared by devices.
- Wipe the buffer after use with `ForceZero()` or the platform equivalent.
- A per-device key is preferable to one key copied to every device.

The expiry timestamps come from `LowResTimer()`, so provide a stable clock
across the restart. If the platform clock restarts from zero, the restored
expiry values may not be meaningful; rotate the keys immediately after boot
and accept that pre-reboot tickets can no longer resume.

## Memory and concurrency

The default key context adds two key slots, one RNG and one context mutex to
the server `WOLFSSL_CTX`. It does not add per-connection ticket-encryption
state. Tickets themselves are sized by
`SESSION_TICKET_STATIC_LEN`/`SESSION_TICKET_LEN`; `WOLFSSL_TICKET_EXTRA_PADDING_SZ`
controls spare space for the encryption callback.

On a threaded build, key selection and copying are protected by the context
mutex. Encryption and decryption run outside the lock on a copy of the selected
key, so a concurrent rotation cannot replace a key while it is in use. With
`SINGLE_THREADED`, no mutex is allocated or used.

## Verification

The API and TLS 1.3 rotation tests are in `tests/api/test_session.c` and
`tests/api/test_tls13.c`:

- `test_wolfSSL_CTX_rotate_ticket_keys`
- `test_tls12_ticket_key_rotation_resume`
- `test_tls13_ticket_key_rotation_resume`

Run the unit tests with session tickets and TLS 1.3 enabled:

```sh
./configure --enable-session-ticket --enable-tls13
make -j
./tests/unit.test --group session --group tls13
```
