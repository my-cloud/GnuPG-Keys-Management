---
layout: default
title: "YubiKey Setup"
permalink: /YubiKey-Setup/
---
## YubiKey Setup

Initialize and harden the YubiKey before copying real OpenPGP subkeys onto it.
A YubiKey contains several independent applications with separate credentials:

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

This repository primarily uses the OpenPGP application with GnuPG. PIV is
covered separately so that an enabled application is not unintentionally left
with default administrative credentials.

**Do not confuse the OpenPGP Admin PIN with the PIV Management Key.** They
belong to different applications and are not interchangeable.

### Inspect the YubiKey first

Start with non-destructive inspection:

```bash
ykman list
ykman info
ykman openpgp info
ykman piv info
gpg --card-status
```

They verify different parts of the device:

- `ykman list` identifies connected YubiKeys and their serial numbers.
- `ykman info` shows the model, firmware, and enabled USB applications/interfaces.
- `ykman openpgp info` shows OpenPGP configuration and retry state.
- `ykman piv info` shows PIV credentials, slots, and retry state.
- `gpg --card-status` confirms that GnuPG can see the OpenPGP application.

Record the model and firmware: firmware determines which PIV management-key
algorithms are supported.

Do not reset an application just because the YubiKey is being initialized.
These commands are destructive and are **not** part of this setup:

```bash
ykman openpgp reset
ykman piv reset
```

They wipe the corresponding application's keys and configuration. If the card
is not known to be empty, stop and investigate instead.

### OpenPGP credentials

The OpenPGP application has three credentials:

| Credential | Purpose |
| --- | --- |
| User PIN | Authorizes normal use of OpenPGP private keys |
| Reset Code | Can recover/change a blocked User PIN |
| Admin PIN | Controls OpenPGP administration |

The factory User PIN is `123456`, the factory Admin PIN is `12345678`, and the
Reset Code is initially unset. Replace the defaults before installing real
keys. Use interactive entry so the new values do not appear in shell history:

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

Use this initialization order:

1. Change the User PIN with menu item 1.
2. Change the Admin PIN with menu item 3.
3. Configure a Reset Code with menu item 4.
4. Configure the retry counters as described next.

Do not put private PIN or Reset Code values in commands, scripts, or notes that
are not protected as recovery material.

### OpenPGP retry counters

GnuPG displays the counters in this order:

```text
User PIN / Reset Code / Admin PIN
```

For this walkthrough, configure 12, 3, and 6 attempts respectively:

```bash
ykman openpgp access set-retries 12 3 6
```

```text
12 = User PIN attempts
 3 = Reset Code attempts
 6 = Admin PIN attempts
```

`gpg --card-edit` and `gpg --card-status` display these values, but `ykman` is
used to configure their maxima. Verify the result:

```bash
ykman openpgp info
gpg --card-status
```

Expected GnuPG output includes:

```text
PIN retry counter : 12 3 6
```

### OpenPGP touch policy

The OpenPGP application has three operational key slots:

```text
sig = signing
dec = decryption slot used by the encryption subkey
aut = authentication
```

Require physical touch for each slot:

```bash
ykman openpgp keys set-touch sig on
ykman openpgp keys set-touch dec on
ykman openpgp keys set-touch aut on
```

Current `ykman` calls the OpenPGP card slot `dec`; it holds the `[E]` subkey and
performs decryption. Some older documentation calls the same slot `enc`.

Writing a key to a slot can reset that slot's touch policy. Repeat these three
commands and verify `ykman openpgp info` immediately after the `keytocard`
procedure too.

Available policies include:

```text
off    = no touch required
on     = touch required
cached = one touch can be cached briefly
fixed  = touch required and cannot later be disabled without replacing/deleting the key
```

This walkthrough uses `on`. Do not choose `fixed` casually: its irreversible
semantics can complicate recovery or a later policy change.

PIN and touch address different checks:

```text
PIN   -> proves knowledge of the credential
Touch -> proves physical presence of the YubiKey
```

### PIV credentials

PIV is separate from OpenPGP:

| PIV credential | Purpose |
| --- | --- |
| PIN | Authorizes use of PIN-protected PIV private keys |
| PUK | Recovers/unblocks the PIV PIN |
| Management Key | Authorizes administrative modification of PIV |

The factory PIV PIN is `123456`, the factory PUK is `12345678`, and the factory
management key is a published, device-wide default. They are initialization
values, not credentials to keep for real use.

The practical threat model is:

```text
PIN
→ can authorize use of PIN-protected PIV credentials.

PUK
→ can recover/reset the PIN.

Management Key
→ can administer/reprovision PIV, but does not by itself extract
  an existing private key or bypass a PIN-protected key operation.
```

A stolen YubiKey that still uses the default management key can be
administratively modified and its PIV credentials can be overwritten. The
management key is important, but it is not equivalent to the PIN; possession of
it does not automatically let someone impersonate the owner with an existing
PIN-protected private key.

### PIV retry counters

The command syntax is:

```bash
PIN_RETRIES="<CHOOSE_PIN_RETRIES>"
PUK_RETRIES="<CHOOSE_PUK_RETRIES>"
ykman piv access set-retries "$PIN_RETRIES" "$PUK_RETRIES"
```

> **Warning:** changing the PIV retry-counter maxima resets the PIV PIN and PUK
> to their factory defaults.

If you want custom maxima, set them before choosing final credentials:

```text
1. Configure PIV retry counters
2. Change PIV PIN
3. Change PIV PUK
4. Configure PIV management key
```

There is no mandatory PIN/PUK retry policy for this repository. Choose values
that fit your threat model and recovery plan.

### Change the PIV PIN and PUK

Use interactive commands so credentials are not placed directly in shell
history:

```bash
ykman piv access change-pin
ykman piv access change-puk
```

Treat the PUK as a recovery credential, not something used routinely. Store it
according to your recovery plan.

### PIV Management Key

The PIV Management Key is an administrative cryptographic key, not a human
password. Choose one of the following management models; they are alternatives,
not two required steps.

#### Option A — externally stored management key

For an older YubiKey that supports only TDES, generate 24 random bytes (48
hexadecimal characters):

```bash
openssl rand -hex 24
ykman piv access change-management-key \
    --algorithm tdes
```

Paste the generated value at the interactive prompt. For an AES-256-capable
YubiKey, generate 32 random bytes (64 hexadecimal characters):

```bash
openssl rand -hex 32
ykman piv access change-management-key \
    --algorithm aes256
```

Store the resulting management key securely. It is required for future PIV
administration. Do not put it directly on the command line.

#### Option B — PIN-protected management key on the YubiKey

Let `ykman` generate a random management key, store it on the YubiKey, and
protect access to it with the PIV PIN:

```bash
ykman piv access change-management-key --protect
```

Conceptually:

```text
PIV PIN
   ↓
protected management-key storage
   ↓
PIV administrative operation
```

This avoids retaining the raw management key manually. It also means the raw
management key is not another secret that the user must remember.

### Firmware-specific management-key algorithms

Check `ykman info` before selecting an algorithm. Standard YubiKey 5 firmware
5.3.x and older supports TDES only; AES-128, AES-192, and AES-256 management
keys are supported starting with firmware 5.4.2. Firmware 5.7 changed the
standard YubiKey default from TDES to AES-192. AES-256 may be selected explicitly
when the device supports it. Some FIPS models impose additional restrictions,
so follow the device-specific Yubico guidance.

For a legacy device such as firmware 5.1.2, a reasonable protected setup is:

```bash
ykman piv access change-management-key \
    --algorithm tdes \
    --touch \
    --protect
```

For an AES-capable personal YubiKey, this walkthrough prefers:

```bash
ykman piv access change-management-key \
    --algorithm aes256 \
    --touch \
    --protect
```

The flags mean:

```text
--protect
    Generate/store the management key on the YubiKey, protected by PIN.

--touch
    Require physical touch when authenticating with the management key
    for an administrative operation.
```

With both flags, the effective flow is:

```text
PIN
 ↓
retrieve/use protected random management key
 ↓
physical touch
 ↓
PIV administration
```

Standard YubiKey firmware is not field-upgradable. Do not try to upgrade it as
part of this procedure; choose settings supported by the installed firmware.

### Verify the final device state

Finish with non-destructive inspection:

```bash
ykman info
ykman openpgp info
ykman piv info
gpg --card-status
```

Checklist:

```text
[ ] OpenPGP User PIN changed
[ ] OpenPGP Admin PIN changed
[ ] OpenPGP Reset Code configured
[ ] OpenPGP retry counters configured
[ ] OpenPGP touch policies configured
[ ] PIV PIN changed, if PIV is used
[ ] PIV PUK changed, if PIV is used
[ ] PIV management key is no longer unintentionally using the default
[ ] Device firmware / capabilities recorded
```

Do not record actual PINs, PUKs, Reset Codes, or management keys in this
checklist.

### Continue to GPG subkey installation

The device is now ready for the
[Protect your subkeys](./Technical-Walkthrough.md#protect-your-subkeys)
procedure. That section handles `keytocard`; this page only initializes and
hardens the physical YubiKey.

```text
Create master key
      ↓
Create S / E / A subkeys
      ↓
Protect/offline master key
      ↓
YubiKey Setup
      ↓
Move subkeys with keytocard
      ↓
Test the key
```

### Yubico references

- [YubiKey Manager OpenPGP commands](https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html)
- [YubiKey Manager PIV commands](https://docs.yubico.com/software/yubikey/tools/ykman/PIV_Commands.html)
- [YubiKey 5 firmware 5.7 specifics](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-firmware-5.7.html)
- [YubiKey OpenPGP specifics and defaults](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-apps-openpgp.html)
- [YubiKey PIV specifics and defaults](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/yk5-apps-piv.html)
