---
layout: default
title: "GnuPG Key Technical Walkthrough"
---
## GnuPG Key Technical Walkthrough

Create a master key with Certification capability and then 3 subkeys with these capabilities:  Sign, Encrypt ant Authenticate.

Then for increasing the secutity, the master private key must be stored on an encrypted storage device and deleted from the computer.

The subkeys can also be offloaded to a smart card for going an extra-mile in the GPG keys protection.

---

- **[Create gpg key pair](./Technical-Walkthrough.md#create-gpg-key-pair)**
- **[Create Subkeys](./Technical-Walkthrough.md#create-subkeys)**
- **[Key Rotation? Sign this new key ](./Technical-Walkthrough.md#key-rotation-sign-this-new-key)**
- **[Protect your master key](./Technical-Walkthrough.md#protect-your-master-key)**
- **[Use the offline master key](./Technical-Walkthrough.md#use-the-offline-master-key)**
- **[Protect your subkeys](./Technical-Walkthrough.md#protect-your-subkeys)**
- **[Test your key](./Technical-Walkthrough.md#test-your-key)**
- **[Import / Re-import](./Technical-Walkthrough.md#import--re-import)**
- **[Deletion / Revocation](./Technical-Walkthrough.md#deletion--revocation)**
- **[References](./Technical-Walkthrough.md#references)**

---

### Create gpg key pair

create a template:

```bash
cat << EOF > primary_gpg_key_constructor.tpl
%echo Generating a RSA key 8192
Key-Type: RSA
Key-Length: 8192
#Key-usage can be cert,sign,auth
Key-Usage: cert
Name-Real: My Name
#Name-Comment:
Name-Email: something@something
Expire-Date: 0
# Do a commit here, so that we can later print "done" :-)
%commit
%echo done
EOF
```

GnuPG asks for the passphrase through pinentry. Do not put a real passphrase in
the template, command line, environment, or shell history.



#### generate the key

`gpg --batch --generate-key  --enable-large-rsa primary_gpg_key_constructor.tpl`

(`enable-large-rsa` is to have a key of size over 4096 bits)

#### Check your private keys

`gpg --list-secret-keys`

```
sec   rsa8192 2023-01-14 [C]
      E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
uid           [ultimate] My Name <something@something>
```



#### Check your public keys

`gpg --list-keys`

```
pub   rsa8192 2023-01-14 [C]
      E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
uid           [ultimate] My Name <something@something>
```



### Create Subkeys

Edit your key 

`gpg --expert --edit-key E6B0933DD47E3B2F823334F0A567B3B17C27BF6D`

then add a key 

```
gpg> addkey
Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 
```

##### Signature key

To create a RSA 4096 bits sign key:
Select 4 , set the bit size, the expiration date and validate.

##### Encryption key

To create a RSA 4096 bits encryption key:
Add another key with `addkey`, select 6, set the bit size, the expiration date and validate.

##### Authentication key

To create a RSA 4096 bits authentication key:
Add the last key with `addkey`, select 8 then notice the `Current allowed actions` is already set to `Sign Encrypt` 
Unset these capabilities by selecting `S` and `E`
Then set the authenticate capability by selecting `A`

then `Q` to finish, set the bit size, the expiration date and validate.

Then exit gpg nicely by entering `q` then `y`

#### Check all the Subkeys

` gpg --fingerprint --fingerprint E6B0933DD47E3B2F823334F0A567B3B17C27BF6D`

the `--fingerprint` twice is to reveal the subkeys information
```
pub   rsa8192 2023-01-14 [C]
      E6B0 933D D47E 3B2F 8233  34F0 A567 B3B1 7C27 BF6D
uid           [ultimate] My Name <something@something>
sub   rsa4096 2023-01-14 [S] [expires: 2030-01-12]
      61DF 146E F7F6 0971 5E60  46B8 8C75 87C8 6EAC 2F2F
sub   rsa4096 2023-01-14 [E] [expires: 2030-01-12]
      DBBC AE3F 8166 9FF4 75B7  5F35 51D8 8833 7727 23AD
sub   rsa4096 2023-01-14 [A] [expires: 2030-01-12]
      F4CF F234 E969 2304 CC58  D55B D13A E6B6 5E8B E723
```

### Key Rotation? Sign this new key 

In case it is not your first key and you are doing a key rotation:  you can sign the new master key with your previous key that you're transitioning from.

`gpg --sign-key <FULL_PRIMARY_KEY_FINGERPRINT>`

---

### Protect your master key

The certification-capable primary key is kept offline. The normal workstation
retains its public certificate and operational subkeys, but not the primary
secret key. Before removing anything, make a complete backup on mounted,
encrypted external storage and prove that the backup can be restored.

In these commands, `EXPORT_FINGERPRINT` is the full fingerprint of the primary
OpenPGP key. A fingerprint identifies an OpenPGP key; a shorter key ID is only a
convenient, collision-prone abbreviation. A keygrip is a different identifier
used internally by `gpg-agent` for one secret-key component.

The hardened snippets in this section use Bash syntax. Run each procedure in
Bash and keep its related code blocks in the same shell session.

#### Export your master key

Set the mount and key details carefully. `mountpoint` prevents the script from
silently creating the expected USB path on the unencrypted root filesystem when
the encrypted device is not mounted. It proves that the path is a mount point,
not that the filesystem is encrypted; confirm the configured path belongs to
the unlocked encrypted device.

The script refuses to overwrite any existing recovery artifact. Preserve the
known-good backup until a new export has passed the independent restore test.

```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077

ENCRYPTED_MOUNT="/run/media/user/Encrypted_Device"
EXPORT_DIR="$ENCRYPTED_MOUNT/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"

if ! mountpoint -q -- "$ENCRYPTED_MOUNT"; then
    echo "Encrypted storage is not mounted at $ENCRYPTED_MOUNT" >&2
    exit 1
fi

if [[ ! -d "$ENCRYPTED_MOUNT" || ! -w "$ENCRYPTED_MOUNT" ]]; then
    echo "Encrypted storage is not a writable directory" >&2
    exit 1
fi

# Creating this directory is safe only after validating the mount above.
install -d -m 0700 -- "$EXPORT_DIR"

artifacts=(
    "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"
    "$EXPORT_DIR/$EXPORT_FILENAME.subkeys.sec"
    "$EXPORT_DIR/$EXPORT_FILENAME.pubkey.asc"
    "$EXPORT_DIR/$EXPORT_FILENAME.keygrip.txt"
    "$EXPORT_DIR/gnupg-ownertrust.txt"
    "$EXPORT_DIR/revoke.no-reason.asc"
    "$EXPORT_DIR/revoke.compromised.asc"
    "$EXPORT_DIR/revoke.superseded.asc"
    "$EXPORT_DIR/revoke.no-longer-used.asc"
)

for artifact in "${artifacts[@]}"; do
    if [[ -e "$artifact" ]]; then
        echo "Refusing to overwrite existing artifact: $artifact" >&2
        exit 1
    fi
done

gpg --armor \
    --output "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec" \
    --export-secret-keys "$EXPORT_FINGERPRINT"
gpg --armor \
    --output "$EXPORT_DIR/$EXPORT_FILENAME.subkeys.sec" \
    --export-secret-subkeys "$EXPORT_FINGERPRINT"
gpg --armor \
    --output "$EXPORT_DIR/$EXPORT_FILENAME.pubkey.asc" \
    --export "$EXPORT_FINGERPRINT"
gpg --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT" \
    > "$EXPORT_DIR/$EXPORT_FILENAME.keygrip.txt"
gpg --export-ownertrust > "$EXPORT_DIR/gnupg-ownertrust.txt"

declare -A REVOCATION_REASONS=(
    [no-reason]=0
    [compromised]=1
    [superseded]=2
    [no-longer-used]=3
)

for reason in no-reason compromised superseded no-longer-used; do
    output="$EXPORT_DIR/revoke.$reason.asc"
    printf '%s\n' "y" "${REVOCATION_REASONS[$reason]}" "" "y" |
        gpg --no-tty --command-fd 0 --armor --output "$output" \
            --gen-revoke "$EXPORT_FINGERPRINT"
done

chmod 0600 -- "${artifacts[@]}"
```

`--command-fd` supplies the initial confirmation, revocation reason, an empty
description, and the final confirmation. `--no-tty` keeps those scripted answers
on that channel in a headless shell. Pinentry can still request the primary-key
passphrase securely; never put that passphrase in this script, the environment,
a command line, or shell history.

The reason-code mapping is:

```text
0 = no reason specified
1 = key compromised
2 = key superseded
3 = key no longer used
```

Any one of these certificates can revoke the entire primary key. Keeping four
does not provide four levels of revocation; it lets you publish the certificate
with the most accurate reason later.

The recovery bundle contains:

| File | Purpose |
| --- | --- |
| `*.private-key.sec` | Complete disaster-recovery export: primary and subkey secret material |
| `*.subkeys.sec` | Secret-subkeys-only export for operational recovery |
| `*.pubkey.asc` | Convenient standalone public certificate |
| `*.keygrip.txt` | Diagnostic mapping between OpenPGP keys and agent keygrips |
| `gnupg-ownertrust.txt` | Local GnuPG ownertrust database |
| `revoke.*.asc` | Independent primary-key revocation certificates |

The older name `pgp-ownertrust.asc` referred to the same output from
`gpg --export-ownertrust`. It is now called `gnupg-ownertrust.txt` because the
file is plain-text GnuPG ownertrust data, not an ASCII-armored OpenPGP object.
Restore it with:

```bash
gpg --import-ownertrust "$EXPORT_DIR/gnupg-ownertrust.txt"
```

Protect every file in this directory. The private exports and revocation
certificates are especially sensitive; access to either secret keys or a
revocation certificate has serious consequences.

Do not assume the active home is `$HOME/.gnupg`. Inspect it with:

```bash
gpgconf --list-dirs homedir
```

This matters when `GNUPGHOME` points into `/dev/shm`. An automatically generated
`openpgp-revocs.d/<fingerprint>.rev` normally appears when a primary key is first
created. Importing an existing secret key into a new home does not necessarily
recreate it. Consequently, `$HOME/.gnupg/openpgp-revocs.d/` is the wrong path
for a temporary home, and the RAM keyring must not depend on that file. Treat the
explicit `revoke.*.asc` files on encrypted storage as persistent, independent
recovery artifacts.

#### Validate the complete backup

Do this before deleting any key material. A successful export command alone is
not proof: a source keyring containing only unavailable or smartcard stubs can
produce an unusable result.

```bash
set -euo pipefail
umask 077

EXPORT_DIR="/run/media/user/Encrypted_Device/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"
TEST_GNUPGHOME="$(mktemp -d /dev/shm/gnupg-test.XXXXXX)"
chmod 0700 "$TEST_GNUPGHOME"

cleanup_test_gnupghome() {
    if [[ -n "${TEST_GNUPGHOME:-}" && -d "$TEST_GNUPGHOME" &&
          ! -L "$TEST_GNUPGHOME" &&
          "$TEST_GNUPGHOME" == /dev/shm/gnupg-test.* ]]; then
        gpgconf --homedir "$TEST_GNUPGHOME" --kill all || true
        rm -rf -- "$TEST_GNUPGHOME"
        unset TEST_GNUPGHOME
    else
        echo "Refusing to delete suspicious TEST_GNUPGHOME" >&2
        return 1
    fi
}
trap cleanup_test_gnupghome EXIT

gpg --homedir "$TEST_GNUPGHOME" \
    --import "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"

gpg --homedir "$TEST_GNUPGHOME" \
    --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT"

mapfile -t secret_statuses < <(
    LC_ALL=C gpg --homedir "$TEST_GNUPGHOME" \
        --list-secret-keys "$EXPORT_FINGERPRINT" 2>/dev/null |
        awk '/^(sec|ssb)/{print $1}'
)

validation_failed=0
if [[ "${secret_statuses[0]:-}" != "sec" ||
      "${#secret_statuses[@]}" -lt 4 ]]; then
    validation_failed=1
fi
for key_status in "${secret_statuses[@]}"; do
    if [[ "$key_status" != "sec" && "$key_status" != "ssb" ]]; then
        validation_failed=1
    fi
done

cleanup_test_gnupghome
trap - EXIT

if (( validation_failed != 0 )); then
    echo "Backup validation failed: expected sec plus three usable ssb entries" >&2
    exit 1
fi
```

The listing in the test home must show `sec`, not `sec#` or `sec>`. This
architecture's complete export must also show at least the three operational
subkeys as plain `ssb` entries.

For additional diagnosis, `gpg --list-packets` can inspect an export. Actual
protected secret packets are different from stub representations such as
`gnu-dummy S2K` (unavailable material) and `gnu-divert-to-card S2K` (smartcard
material). Packet inspection is useful, but the isolated test import remains
the required validation.

#### Master key deletion

**Destructive step:** continue only after the complete backup has passed the
test above and the encrypted storage has been safely unmounted or disconnected.
The preferred modern command deletes only the primary secret component:

```bash
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"

gpg --delete-secret-keys "${EXPORT_FINGERPRINT}!"
gpg --list-secret-keys "$EXPORT_FINGERPRINT"
```

The quoted `!` is an exact-key selector. With the full primary fingerprint it
tells GnuPG to delete only the primary secret component, instead of the primary
and all its secret subkeys. Review GnuPG's confirmation prompt carefully.

The lower-level alternative is `gpg-connect-agent DELETE_KEY`. `DELETE_KEY`
expects a **keygrip**, not a key ID, long key ID, or OpenPGP fingerprint:

```bash
gpg --list-secret-keys --with-keygrip "$EXPORT_FINGERPRINT"
gpg-connect-agent "DELETE_KEY <PRIMARY_SEC_KEYGRIP>" /bye
gpg --list-secret-keys "$EXPORT_FINGERPRINT"
```

Use only the keygrip printed directly under the primary `sec` entry. Do not use
a subkey keygrip.

The normal workstation should now show the offline primary and locally retained
subkeys:

```text
sec#  ... [C]
ssb   ... [S]
ssb   ... [E]
ssb   ... [A]
```

After the subkeys have been moved to a YubiKey, the expected state is:

```text
sec#  ... [C]
ssb>  ... [S]
ssb>  ... [E]
ssb>  ... [A]
```


### Use the offline master key

Restore the complete backup into a new RAM-backed GnuPG home for maintenance.
This environment is isolated: do not point `--keyring` at the normal
workstation's `pubring.kbx`. Import everything needed into the temporary home.
Run this whole procedure in one Bash session so its cleanup trap remains active.

First verify the encrypted mount as in the export procedure, then create the
temporary keyring:

```bash
set -euo pipefail
umask 077

ENCRYPTED_MOUNT="/run/media/user/Encrypted_Device"
EXPORT_DIR="$ENCRYPTED_MOUNT/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"

if ! mountpoint -q -- "$ENCRYPTED_MOUNT"; then
    echo "Encrypted storage is not mounted at $ENCRYPTED_MOUNT" >&2
    exit 1
fi

ORIGINAL_GNUPGHOME_SET=0
ORIGINAL_GNUPGHOME=""
if [[ -v GNUPGHOME ]]; then
    ORIGINAL_GNUPGHOME_SET=1
    ORIGINAL_GNUPGHOME="$GNUPGHOME"
fi
NORMAL_GNUPGHOME="$(gpgconf --list-dirs homedir)"
printf 'Normal GnuPG home: %s\n' "$NORMAL_GNUPGHOME"

GNUPGHOME="$(mktemp -d /dev/shm/gnupg.XXXXXX)"
export GNUPGHOME
chmod 0700 "$GNUPGHOME"

cleanup_offline_gnupghome() {
    if [[ -n "${GNUPGHOME:-}" && -d "$GNUPGHOME" && ! -L "$GNUPGHOME" &&
          "$GNUPGHOME" == /dev/shm/gnupg.* ]]; then
        gpgconf --homedir "$GNUPGHOME" --kill all || true
        rm -rf -- "$GNUPGHOME"
        unset GNUPGHOME
    else
        echo "Refusing to delete suspicious GNUPGHOME" >&2
        return 1
    fi

    if (( ORIGINAL_GNUPGHOME_SET != 0 )); then
        export GNUPGHOME="$ORIGINAL_GNUPGHOME"
    fi
}
trap cleanup_offline_gnupghome EXIT

gpg --import "$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"
gpg --import-ownertrust "$EXPORT_DIR/gnupg-ownertrust.txt"
gpg --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT"
```

GnuPG's secret-key indicators mean:

| Indicator | Meaning |
| --- | --- |
| `sec` | Actual primary secret key is available |
| `sec#` | Primary secret key is unavailable/offline |
| `ssb` | Actual secret subkey is locally available |
| `ssb#` | Secret subkey is unavailable |
| `ssb>` | Secret subkey is represented by a smartcard/YubiKey stub |

The restored primary must be `sec` in this isolated home. A complete backup that
contains the subkey secrets should similarly show actual `ssb` entries. Stop if
the primary is `sec#`: the backup did not restore usable primary secret material.

Perform primary-key maintenance entirely inside this home, for example:

```bash
gpg --edit-key "$EXPORT_FINGERPRINT"
```

This includes changing the primary passphrase, extending subkey expiration,
adding or revoking subkeys, modifying identities, and making certifications.

#### Re-export after maintenance

Changes made in RAM do **not** update the encrypted `.private-key.sec` file.
Create a new candidate without overwriting the known-good backup:

```bash
NEW_BACKUP="$EXPORT_DIR/$EXPORT_FILENAME.private-key.NEW.sec"

if [[ -e "$NEW_BACKUP" ]]; then
    echo "Refusing to overwrite existing candidate: $NEW_BACKUP" >&2
    exit 1
fi

gpg --armor \
    --output "$NEW_BACKUP" \
    --export-secret-keys "$EXPORT_FINGERPRINT"
chmod 0600 "$NEW_BACKUP"
```

Test the candidate in a second, independent temporary home while the maintenance
home is still available:

```bash
NEW_TEST_GNUPGHOME="$(mktemp -d /dev/shm/gnupg-test.XXXXXX)"
chmod 0700 "$NEW_TEST_GNUPGHOME"

cleanup_new_test_gnupghome() {
    if [[ -n "${NEW_TEST_GNUPGHOME:-}" && -d "$NEW_TEST_GNUPGHOME" &&
          ! -L "$NEW_TEST_GNUPGHOME" &&
          "$NEW_TEST_GNUPGHOME" == /dev/shm/gnupg-test.* ]]; then
        gpgconf --homedir "$NEW_TEST_GNUPGHOME" --kill all || true
        rm -rf -- "$NEW_TEST_GNUPGHOME"
        unset NEW_TEST_GNUPGHOME
    else
        echo "Refusing to delete suspicious NEW_TEST_GNUPGHOME" >&2
        return 1
    fi
}
trap 'cleanup_new_test_gnupghome; cleanup_offline_gnupghome' EXIT

gpg --homedir "$NEW_TEST_GNUPGHOME" --import "$NEW_BACKUP"
gpg --homedir "$NEW_TEST_GNUPGHOME" \
    --list-secret-keys --fingerprint --with-keygrip "$EXPORT_FINGERPRINT"

mapfile -t new_secret_statuses < <(
    LC_ALL=C gpg --homedir "$NEW_TEST_GNUPGHOME" \
        --list-secret-keys "$EXPORT_FINGERPRINT" 2>/dev/null |
        awk '/^(sec|ssb)/{print $1}'
)

new_validation_failed=0
if [[ "${new_secret_statuses[0]:-}" != "sec" ||
      "${#new_secret_statuses[@]}" -lt 4 ]]; then
    new_validation_failed=1
fi
for key_status in "${new_secret_statuses[@]}"; do
    if [[ "$key_status" != "sec" && "$key_status" != "ssb" ]]; then
        new_validation_failed=1
    fi
done

cleanup_new_test_gnupghome
trap cleanup_offline_gnupghome EXIT

if (( new_validation_failed != 0 )); then
    echo "Candidate validation failed: keeping the known-good backup" >&2
    exit 1
fi
```

Only after that succeeds, preserve the old backup and promote the candidate.
Both moves stay on the encrypted filesystem:

```bash
KNOWN_GOOD="$EXPORT_DIR/$EXPORT_FILENAME.private-key.sec"
ARCHIVE="$KNOWN_GOOD.previous.$(date -u +%Y%m%dT%H%M%SZ)"

if [[ ! -f "$KNOWN_GOOD" || -e "$ARCHIVE" ]]; then
    echo "Refusing unsafe backup replacement" >&2
    exit 1
fi

cp --preserve=mode,timestamps -- "$KNOWN_GOOD" "$ARCHIVE"
mv -- "$NEW_BACKUP" "$KNOWN_GOOD"
chmod 0600 -- "$KNOWN_GOOD" "$ARCHIVE"
```

Keep the archived known-good backup until the promoted copy has been verified
and the encrypted storage itself is backed up. If maintenance changed public
key data or subkeys, also refresh the public and subkeys-only exports using the
same create-new, test, archive, and promote pattern.

#### Clean up the RAM keyring

Check the path before recursive deletion:

```bash
cleanup_offline_gnupghome
trap - EXIT

CURRENT_GNUPGHOME="$(gpgconf --list-dirs homedir)"
printf 'Current GnuPG home: %s\n' "$CURRENT_GNUPGHOME"
if [[ "$CURRENT_GNUPGHOME" != "$NORMAL_GNUPGHOME" ]]; then
    echo "GnuPG home was not restored correctly" >&2
    exit 1
fi
```

The last command must show the normal GnuPG home again. Verify that the normal
workstation still shows `sec#` and its intended `ssb`, `ssb#`, or `ssb>` state.

The complete lifecycle is:

```text
Encrypted offline storage
├── complete primary + subkeys secret backup
├── subkeys-only backup
├── public certificate
├── revocation certificates
├── keygrip/reference information
└── ownertrust
          │ temporary restore
          ▼
RAM-backed GNUPGHOME
├── sec   [C]
├── ssb   [S]
├── ssb   [E]
└── ssb   [A]
          │ maintenance, re-export, independent validation
          ▼
destroy RAM GNUPGHOME

Normal workstation
├── sec#  [C]    primary secret key offline
└── ssb>          operational subkeys on YubiKey
```

---

### Protect your subkeys

You don't want any of your keys on your computer? Dear paranoid friend, I hear
you! An OpenPGP smartcard is one good approach.

Before moving subkeys to the smartcard, initialize and harden the
[YubiKey](./YubiKey-Setup.md). That page covers the independent OpenPGP and PIV
credentials, retry counters, touch policies, and firmware-specific choices.

After setup, inspect the card and then move each subkey:

```bash
gpg --card-status
gpg --edit-key E6B0933DD47E3B2F823334F0A567B3B17C27BF6D
```

```text
gpg> key 1
gpg> keytocard
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]

Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1
```

remember to **unselect** your current subkey before selecting a new one by entering `key 1` again

Perform the same operation to move the other keys to your smartcard, then enter
`save`.

Writing a subkey can reset that slot's touch policy. Reapply the hardened policy
after all three transfers and verify it:

```bash
ykman openpgp keys set-touch sig on
ykman openpgp keys set-touch dec on
ykman openpgp keys set-touch aut on
ykman openpgp info
```

Here is what the result should be:

```
gpg --card-status  
Reader ...........: 0000:0000:A:0
Application ID ...: 0000000000000000000000000000
Application type .: OpenPGP
Version ..........: 2.0
Manufacturer .....: Yubico
Serial number ....: 00000000
Name of cardholder: My Name
Language prefs ...: en
Salutation .......: 
URL of public key : <Whatever PGP server on which you uploaded your key>
Login data .......: Something
Signature PIN ....: forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 127 127 127
PIN retry counter : 12 3 6
Signature counter : 144
Signature key ....: 61DF 146E F7F6 0971 5E60  46B8 8C75 87C8 6EAC 2F2F
      created ....: 2020-01-12 10:09:27
Encryption key....: DBBC AE3F 8166 9FF4 75B7  5F35 51D8 8833 7727 23AD
      created ....: 2020-01-12 10:06:54
Authentication key: F4CF F234 E969 2304 CC58  D55B D13A E6B6 5E8B E723
      created ....: 2020-01-12 10:09:58
General key info..: sub  rsa4096/8C7587C86EAC2F2F 2020-01-12 My Name <something@something>
sec#  rsa8192/A567B3B17C27BF6D  created: 2020-01-12  expires: never     
ssb>  rsa4096/51D88833772723AD  created: 2020-01-12  expires: 2030-01-12
                                card-no: 0000 00000000
ssb>  rsa4096/8C7587C86EAC2F2F  created: 2020-01-12  expires: 2030-01-12
                                card-no: 0000 00000000
ssb>  rsa4096/D13AE6B65E8BE723  created: 2020-01-12  expires: 2030-01-12
                                card-no: 0000 00000000

```



#### Yubitouch

To go further in your setup I suggest checking this [YubiTouch repository](https://github.com/a-dma/yubitouch/)


---

### Test your key

##### Encrypt a file

```
gpg --encrypt --sign --armor -r receipient@gmail.com message.txt
```

The `-encrypt` option tells gpg to encrypt the file, and the `--sign` option tells it to sign the file with your details. The `--armor` option tells gpg to create an ASCII file. The `-r` (recipient) option must be followed by the email address of the person you’re sending the file to.

The `message.txt` is the name of the file.

The file is created with the same name as the original, but with “.asc” appended to the file name (ex. `message.asc`)

##### Decrypt a file

`gpg --decrypt coded.asc`

The `coded.asc` is the file received. 

To redirect the output into another file :

`gpg --decrypt coded.asc > plain.txt`

---

### Import / Re-import

#### Import someone's public key:

```
gpg --import pgp-public-keys.asc
```

The `pgp-public-keys.asc` is the key name of that person.

#### Reimport your own keys

Choose the import according to the target environment. For a normal workstation,
import the public certificate and the secret-subkeys-only export, then restore
the local ownertrust database:

```bash
EXPORT_DIR="/run/media/user/Encrypted_Device/secrets/gpg"
EXPORT_FILENAME="MySelf_Master_MyComment_private_rsa8192"
EXPORT_FINGERPRINT="<FULL_PRIMARY_KEY_FINGERPRINT>"

gpg --import "$EXPORT_DIR/$EXPORT_FILENAME.pubkey.asc"
gpg --import "$EXPORT_DIR/$EXPORT_FILENAME.subkeys.sec"
gpg --import-ownertrust "$EXPORT_DIR/gnupg-ownertrust.txt"
gpg --list-secret-keys "$EXPORT_FINGERPRINT"
```

That workstation import should leave the primary as `sec#`; locally restored
subkeys appear as `ssb`, while YubiKey stubs appear as `ssb>` once associated
with the card.

Import the complete `*.private-key.sec` backup only for disaster recovery or
offline-primary maintenance, preferably in the isolated RAM-backed
`GNUPGHOME` described in [Use the offline master key](#use-the-offline-master-key).
Importing it into the normal home deliberately brings the primary secret key
back online and breaks the intended workstation state.

---

### Deletion / Revocation

#### You messed up? Delete / Revoke your subkey

Something went wrong at the creation, you can delete a subkey? 
Edit your key 

`gpg --expert --edit-key E6B0933DD47E3B2F823334F0A567B3B17C27BF6D`

select the `subkey` or `user ID` (depending on what went wrong)

`key F2DF28C8CDF28C`

Then delete the key

`deluid`

**If you dont select the key... well you'll loose all of them**

*Same principle applies for `uid` and `deluid`*

#### Deletion of a main key

##### secret key

For the offline-master workflow, use the primary-only procedure in
[Master key deletion](#master-key-deletion). Do not use an unqualified primary
fingerprint here: that can delete the primary and all secret subkeys.

If the intention really is to erase the entire secret identity, verify the full
fingerprint and the tested recovery backup immediately before continuing:

```bash
gpg --list-secret-keys --fingerprint "<FULL_PRIMARY_KEY_FINGERPRINT>"
gpg --delete-secret-keys "<FULL_PRIMARY_KEY_FINGERPRINT>"
```

This deletes all secret subkeys associated with that primary key too.

##### public key

Deleting the public certificate also removes the workstation's record of the
identity. Verify the full fingerprint immediately beforehand:

```bash
gpg --fingerprint "<FULL_PRIMARY_KEY_FINGERPRINT>"
gpg --delete-keys "<FULL_PRIMARY_KEY_FINGERPRINT>"
```

#### Revocation

One of your subkey has been compromised, let everyone know!

**Select the subkey!**

`key F2DF28C8CDF28C`

Then delete the key

`revkey`





### References

https://incenp.org/notes/2015/using-an-offline-gnupg-master-key.html

https://gist.github.com/ageis/5b095b50b9ae6b0aa9bf

https://riseup.net/en/security/message-security/openpgp/gpg-best-practices

https://www.devdungeon.com/content/gpg-tutorial

https://8gwifi.org/docs/gpg.jsp

Yubikey : https://www.youtube.com/watch?v=xGsixSh6sC4
