---
layout: default
title: "YubiKey Setup"
permalink: /YubiKey-Setup/
---
## YubiKey Setup

Initialize and harden the YubiKey before copying OpenPGP subkeys onto it.
Remember that a YubiKey contains several independent applications:

```text
YubiKey
├── OpenPGP
│   ├── User PIN
│   ├── Reset Code
│   └── Admin PIN
│
├── PIV
│   ├── PIN
│   ├── PUK
│   └── Management Key
│
├── FIDO2
├── OATH
└── OTP
```

This walkthrough mainly uses OpenPGP with GnuPG. **Do not confuse the OpenPGP
Admin PIN with the PIV Management Key**: they belong to different applications.
PIV is included here so an enabled application is not accidentally left with
default administrative credentials.

### Inspect the YubiKey first

Start with non-destructive commands:

```bash
ykman list
ykman info
ykman openpgp info
ykman piv info
gpg --card-status
```

- `ykman list` identifies the model and serial number.
- `ykman info` shows firmware and enabled USB applications/interfaces.
- `ykman openpgp info` and `ykman piv info` show each application's state and
  retry counters.
- `gpg --card-status` confirms that GnuPG can use the OpenPGP application.

Firmware matters, especially when choosing a PIV management-key algorithm.
Record it now. Do not reset an application merely as part of setup:

```bash
# Destructive: not part of normal initialization
ykman openpgp reset
ykman piv reset
```

Both reset commands wipe data in their respective application.

### OpenPGP credentials

| Credential | Purpose |
| --- | --- |
| User PIN | Authorizes normal use of OpenPGP private keys |
| Reset Code | Can recover/change a blocked User PIN |
| Admin PIN | Controls the OpenPGP application |

The factory User PIN is `123456`, the factory Admin PIN is `12345678`, and the
Reset Code is initially unset. Replace the defaults before installing real
keys. Use GnuPG's interactive menu so new values do not enter shell history:

```bash
gpg --card-edit
```

```text
gpg/card> admin
gpg/card> passwd

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
```

Recommended order:

1. Change the User PIN.
2. Change the Admin PIN.
3. Configure a Reset Code.
4. Configure retry counters.

Do not put real PINs or the Reset Code in commands or shell history.

### OpenPGP retry counters

The displayed order is `User PIN / Reset Code / Admin PIN`. For this project:

```bash
ykman openpgp access set-retries 5 3 6
```

```text
 5 = User PIN attempts
 3 = Reset Code attempts
 6 = Admin PIN attempts
```

`gpg --card-edit` displays these counters, but `ykman` configures their maximum
values. Verify them:

```bash
ykman openpgp info
gpg --card-status
```

Expected GnuPG output:

```text
PIN retry counter : 5 3 6
```

### OpenPGP touch policy

The three operational slots are:

```text
sig = signing
dec = decryption slot used by the encryption subkey
aut = authentication
```

Current `ykman` uses `dec` for the decryption slot; some older documentation
calls the same slot `enc`.

```bash
ykman openpgp keys set-touch sig on
ykman openpgp keys set-touch dec on
ykman openpgp keys set-touch aut on
```

```text
off    = no touch required
on     = touch required
cached = touch can temporarily be cached
fixed  = cannot later be disabled without replacing/deleting the key
```

This walkthrough prefers `on`. Do not casually choose `fixed` because of its
irreversible semantics. Writing a subkey can reset a slot's policy, so repeat
these commands and verify `ykman openpgp info` after `keytocard` too.

```text
PIN   -> proves knowledge of the credential
Touch -> proves physical presence of the YubiKey
```

### PIV credentials

PIV is independent from OpenPGP:

| PIV credential | Purpose |
| --- | --- |
| PIN | Authorizes use of PIN-protected PIV private keys |
| PUK | Recovers/unblocks the PIV PIN |
| Management Key | Authorizes administrative modification of PIV |

The factory PIV PIN is `123456`, the factory PUK is `12345678`, and the factory
Management Key is a published device-wide default. Replace the credentials you
intend to rely on before using PIV for real keys.

```text
PIN
→ can authorize use of PIN-protected PIV credentials.

PUK
→ can recover/reset the PIN.

Management Key
→ can administer/reprovision PIV, but does not by itself extract
  an existing private key or bypass a PIN-protected key operation.
```

A stolen YubiKey with the default management key can have its PIV configuration
or credentials overwritten. The management key is not equivalent to the PIN,
and possessing it does not automatically let someone impersonate the owner.

### PIV retry counters, PIN, and PUK

```bash
PIN_RETRIES="<CHOOSE_PIN_RETRIES>"
PUK_RETRIES="<CHOOSE_PUK_RETRIES>"
ykman piv access set-retries "$PIN_RETRIES" "$PUK_RETRIES"
```

> **Warning:** changing PIV retry-counter maxima resets the PIV PIN and PUK to
> their factory defaults.

If you customize the counters, use this order:

```text
1. Configure PIV retry counters
2. Change PIV PIN
3. Change PIV PUK
4. Configure PIV management key
```

Choose retry values for your own threat and recovery model. Then use interactive
entry so credentials are not placed directly in shell history:

```bash
ykman piv access change-pin
ykman piv access change-puk
```

Treat the PUK as a recovery credential rather than something used routinely.

### PIV Management Key

The Management Key is an administrative cryptographic key, not a human
password. There are two reasonable, alternative management models.

#### Option A — store it externally

Legacy, TDES-only YubiKeys need 24 random bytes (48 hexadecimal characters):

```bash
openssl rand -hex 24
ykman piv access change-management-key --algorithm tdes
```

AES-256-capable YubiKeys need 32 random bytes (64 hexadecimal characters):

```bash
openssl rand -hex 32
ykman piv access change-management-key --algorithm aes256
```

Paste the generated value at the interactive prompt and store it securely. It
will be needed for future PIV administration.

#### Option B — protect it on the YubiKey

```bash
ykman piv access change-management-key --protect
```

`--protect` generates/stores a random Management Key on the YubiKey and protects
access to it with the PIV PIN. This avoids retaining the raw key manually:

```text
PIV PIN
   ↓
protected management-key storage
   ↓
PIV administrative operation
```

Option A and Option B are alternatives; an externally stored key plus
`--protect` are not both required.

### Firmware and physical touch

Firmware 5.3.x and older supports TDES management keys only, so a legacy device
such as firmware 5.1.2 uses:

```bash
ykman piv access change-management-key \
    --algorithm tdes \
    --touch \
    --protect
```

AES management keys are supported starting with firmware 5.4.2. Firmware 5.7
changed the standard YubiKey default to AES-192; AES-256 can be selected when
supported:

```bash
ykman piv access change-management-key \
    --algorithm aes256 \
    --touch \
    --protect
```

```text
--protect = store the random Management Key on the YubiKey, protected by PIN
--touch   = require physical touch for Management Key authentication
```

With both options, the effective flow is `PIN -> protected random Management
Key -> physical touch -> PIV administration`. They are not three secrets to
remember. Some FIPS models impose extra restrictions, so check their specific
documentation. Standard YubiKey firmware is not field-upgradable; do not try to
upgrade it as part of this setup.

### Verify the final device state

```bash
ykman info
ykman openpgp info
ykman piv info
gpg --card-status
```

```text
[ ] OpenPGP User PIN changed
[ ] OpenPGP Admin PIN changed
[ ] OpenPGP Reset Code configured
[ ] OpenPGP retry counters configured
[ ] OpenPGP touch policies configured
[ ] PIV PIN changed, if PIV is used
[ ] PIV PUK changed, if PIV is used
[ ] PIV Management Key is no longer unintentionally using the default
[ ] Device firmware / capabilities recorded
```

Do not record actual secrets in the checklist.

### Continue to GPG subkey installation

Continue with [Protect your subkeys](./Technical-Walkthrough.md#protect-your-subkeys).
That section contains the `keytocard` procedure; this page is only for YubiKey
initialization and hardening.

```text
Create master key -> Create S/E/A subkeys -> Protect/offline master key
                  -> YubiKey Setup -> keytocard -> Test the key
```

### Yubico references

- [OpenPGP commands](https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html)
- [PIV commands](https://docs.yubico.com/software/yubikey/tools/ykman/PIV_Commands.html)
- [Firmware 5.7 specifics](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-firmware-5.7.html)
- [OpenPGP specifics](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-apps-openpgp.html)
- [PIV specifics](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-apps-piv.html)
